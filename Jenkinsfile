pipeline {
    environment {
     VERSION= "4.3.11"
     myjdbcurl="jdbc:mysql://testbuild"+"${env.BUILD_NUMBER}"+"mysql.marathon.l4lb.thisdcos.directory:3306"
     myjdbcuser="root"
     myjdbcpassword="root"
     MAVEN_OPTS="-Xmx2048m";
   }

    agent any 
  
    stages {
        stage("Checkout") {
            steps {
                checkout scm
            }
        }
  
        stage("Install build tools") {
           steps {   
                sh "curl -LO https://s3-us-west-2.amazonaws.com/mesosphere-demo-others/apache-maven-3.5.2-bin.tar.gz"
	            sh "tar xzf apache-maven-3.5.2-bin.tar.gz"
           }
        }

    	stage("Prepare Environment and Build App") {
            steps {   
                parallel(
                    prepare: {
                        echo "JDBC-URL: "+"$myjdbcurl"
                        sh "cp mysql.template mysql.json"
                        sh "sed -ie 's@__REPLACEME__@test/build"+"${env.BUILD_NUMBER}"+"@g;' mysql.json"
                        sh "curl  -X POST marathon.mesos:8080/v2/apps -d @mysql.json -H 'Content-type: application/json'"
                        // marathon credentialsId: 'dcos-token', id: '/test/build'+"${env.BUILD_NUMBER}"+'/mysql', url: 'http://marathon.mesos:8080', filename: 'mysql.json', forceUpdate: true
                        //sh "while true; do curl -s ://testbuild"+"${env.BUILD_NUMBER}"+"mysql.marathon.l4lb.thisdcos.directory:3306 if [[ "$?" -ne 0 ]]; then break fi sleep 5 done"
                    },
                    build: {
                        sh "./apache-maven-3.5.2/bin/mvn -B -P MySQL -DskipTest package"
                    }
                )
            }
        }
        
        stage("Test") {	
            steps {
                sh "./apache-maven-3.5.2/bin/mvn -B -P MySQL test"
            }
            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }

        stage("Archive") { 
            steps {
            }
        }
   
        stage("Deploy") { 
            steps {
                sh "cp tomcat.template tomcat.json"
                sh "sed -ie 's@__REPLACEME__@test/build"+"${env.BUILD_NUMBER}"+"@g;' tomcat.json"
                sh "sed -ie 's@__REPLACEVERSION__@"+"${VERSION}"+"@g;' tomcat.json"
                sh "sed -ie 's@__REPLACEBUILD__@"+"${env.BUILD_NUMBER}"+"@g;' tomcat.json"
                sh "sed -ie 's@__REPLACEVIP__@testbuild"+"${env.BUILD_NUMBER}"+"@g;' tomcat.json"
                sh "curl  -X POST marathon.mesos:8080/v2/apps -d @tomcat.json -H 'Content-type: application/json'"
            }
        }

        stage("Load Test") {
            steps {
                sh "sleep 60"
                sh "curl -LO https://s3-us-west-2.amazonaws.com/mesosphere-demo-others/apache-jmeter-4.0.tgz"
                sh "cp petclinic_test_plan.template petclinic_test_plan.jmx"
                sh "sed -ie 's@__INSERTHOST__@tomcatbuild"+"${env.BUILD_NUMBER}"+".marathon.l4lb.thisdcos.directory@g;' petclinic_test_plan.jmx"
                sh "tar xzf apache-jmeter-4.0.tgz"
                sh "./apache-jmeter-4.0/bin/jmeter -n -l petclinic.jtl -t petclinic_test_plan.jmx"
            }
            post {
                always {
                    perfReport percentiles: '0,50,90,100', sourceDataFiles: 'petclinic.jtl'
                }              
            }
        }
    }
    post { 
        always { 
            sh "curl  -X PUT marathon.mesos:8080/v2/apps//test/build"+"${env.BUILD_NUMBER}"+"/mysql -d @mysql0.json -H 'Content-type: application/json'"          
            sh "curl  -X PUT marathon.mesos:8080/v2/apps//test/build"+"${env.BUILD_NUMBER}"+"/tomcat -d @tomcat0.json -H 'Content-type: application/json'"          
              // marathon credentialsId: 'dcos-token', id: '/test/build'+"${env.BUILD_NUMBER}"+'/mysql', url: 'http://marathon.mesos:8080', filename: 'mysql0.json', forceUpdate: true
	        archiveArtifacts artifacts: 'target/*.war', fingerprint: true
        }
    }
}
