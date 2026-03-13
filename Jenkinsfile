pipeline {
    agent any

    environment {
        MONITORING_NAMESPACE = "monitoring"
        JENKINS_NAMESPACE    = "jenkins"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
                script {
                    echo "Build: ${BUILD_NUMBER} — Branch: ${env.GIT_BRANCH}"
                }
            }
        }

        // ─────────────────────────────────────────────
        // TRAEFIK
        // ─────────────────────────────────────────────

        stage('Deploy Traefik') {
            when { expression { env.GIT_BRANCH == 'origin/main' } }
            steps {
                withCredentials([
                    file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE')
                ]) {
                    sh '''
                        export KUBECONFIG=$KUBECONFIG_FILE

                        echo "=== Configuration Traefik ==="
                        kubectl apply -f traefik/traefik-config.yaml
                        kubectl apply -f traefik/dashboard-ingress.yaml
                    '''
                }
            }
        }

        // ─────────────────────────────────────────────
        // JENKINS
        // ─────────────────────────────────────────────

        stage('Build Jenkins Image') {
            when { expression { env.GIT_BRANCH == 'origin/main' } }
            steps {
                withCredentials([
                    usernamePassword(credentialsId: 'jenkins-dockerhub', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')
                ]) {
                    sh '''
                        docker login -u $DOCKER_USER -p $DOCKER_PASS
                        docker build -t joanisky/jenkins-with-docker:latest jenkins/
                        docker push joanisky/jenkins-with-docker:latest
                    '''
                }
            }
        }

        stage('Deploy Jenkins') {
            when { expression { env.GIT_BRANCH == 'origin/main' } }
            steps {
                withCredentials([
                    file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE')
                ]) {
                    sh '''
                        export KUBECONFIG=$KUBECONFIG_FILE

                        kubectl apply -k jenkins/
                        kubectl rollout status deployment/jenkins \
                            -n $JENKINS_NAMESPACE \
                            --timeout=300s
                    '''
                }
            }
        }

        // ─────────────────────────────────────────────
        // MONITORING
        // ─────────────────────────────────────────────

        stage('Deploy Monitoring Stack') {
            when { expression { env.GIT_BRANCH == 'origin/main' } }
            steps {
                withCredentials([
                    file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE'),
                    string(credentialsId: 'GRAFANA_ADMIN_USER', variable: 'GRAFANA_USER'),
                    string(credentialsId: 'GRAFANA_ADMIN_PASSWORD', variable: 'GRAFANA_PASS')
                ]) {
                    sh """
                        export KUBECONFIG=\$KUBECONFIG_FILE

                        # Namespace + Ingress + PodMonitors
                        kubectl apply -k monitoring/

                        # Secret Grafana — idempotent
                        kubectl create secret generic kube-prometheus-stack-grafana \
                            --from-literal=admin-user=\${GRAFANA_USER} \
                            --from-literal=admin-password=\${GRAFANA_PASS} \
                            --from-literal=ldap-toml='' \
                            --namespace ${MONITORING_NAMESPACE} \
                            --dry-run=client -o yaml | kubectl apply -f -

                        # Helm repo
                        helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
                        helm repo update

                        # Install / upgrade kube-prometheus-stack
                        helm upgrade --install kube-prometheus-stack \
                            prometheus-community/kube-prometheus-stack \
                            --namespace ${MONITORING_NAMESPACE} \
                            --values monitoring/helm-values.yaml \
                            --set grafana.adminPassword=\${GRAFANA_PASS} \
                            --wait \
                            --timeout 5m
                    """
                }
            }
        }
		// ─────────────────────────────────────────────
		// BACKUPS (Git, Infra, Jenkins)
		// ─────────────────────────────────────────────

		stage('Deploy Backups') {
			when { expression { env.GIT_BRANCH == 'origin/main' } }
			steps {
				withCredentials([
					file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE')
				]) {
					sh '''
						export KUBECONFIG=$KUBECONFIG_FILE

						echo "=== Namespace infra ==="
						kubectl apply -f infra/namespace-infra.yaml

						echo "=== CronJobs backup ==="
						kubectl apply -k cronjobs

						echo "=== CronJobs déployés ==="
						kubectl get cronjobs -n infra
					'''
				}
			}
		}
        // ─────────────────────────────────────────────
        // VÉRIFICATION
        // ─────────────────────────────────────────────

        stage('Verify') {
            when { expression { env.GIT_BRANCH == 'origin/main' } }
            steps {
                withCredentials([
                    file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE')
                ]) {
                    sh '''
                        export KUBECONFIG=$KUBECONFIG_FILE

                        echo "=== Pods Jenkins ==="
                        kubectl get pods -n $JENKINS_NAMESPACE

                        echo "=== Pods Monitoring ==="
                        kubectl get pods -n $MONITORING_NAMESPACE

                        echo "=== Ingress ==="
                        kubectl get ingress -n $JENKINS_NAMESPACE
                        kubectl get ingress -n $MONITORING_NAMESPACE
                        kubectl get ingress -n kube-system
                    '''
                }
            }
        }

        stage('Health Checks') {
            when { expression { env.GIT_BRANCH == 'origin/main' } }
            steps {
                script {
                    // Grafana
                    timeout(time: 3, unit: 'MINUTES') {
                        waitUntil {
                            script {
                                def response = sh(
                                    script: 'curl -4 -s -o /dev/null -w "%{http_code}" https://grafana.jonathanlore.fr/api/health',
                                    returnStdout: true
                                ).trim()
                                if (response == '200') {
                                    echo "✅ Grafana accessible"
                                    return true
                                }
                                echo "⏳ Grafana en attente... (${response})"
                                sleep 10
                                return false
                            }
                        }
                    }

                    // Jenkins
                    timeout(time: 3, unit: 'MINUTES') {
                        waitUntil {
                            script {
                                def response = sh(
									script: 'curl -4 -s -o /dev/null -w "%{http_code}" https://jenkins.jonathanlore.fr/login',
									returnStdout: true
                                ).trim()
                                if (response == '200') {
                                    echo "✅ Jenkins accessible"
                                    return true
                                }
                                echo "⏳ Jenkins en attente... (${response})"
                                sleep 10
                                return false
                            }
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
        success {
            echo "✅ Infrastructure déployée avec succès !"
            echo "🔧 Jenkins  : https://jenkins.jonathanlore.fr"
            echo "📊 Grafana  : https://grafana.jonathanlore.fr"
            echo "🌐 Traefik  : https://traefik.jonathanlore.fr"
        }
        failure {
            echo "❌ Pipeline infra échoué — vérifier les logs ci-dessus"
        }
    }
}