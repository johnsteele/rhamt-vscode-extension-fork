#!/usr/bin/env groovy

node('rhel8'){
	stage('Checkout repo') {
		deleteDir()
		// git url: 'https://github.com/windup/rhamt-vscode-extension.git'
		git url: 'https://github.com/windup/rhamt-vscode-extension.git',
			branch: "${BRANCH}"
	}

	stage('Install requirements') {
        def nodeHome = tool 'nodejs-lts'
        env.PATH="${env.PATH}:${nodeHome}/bin"
        // sh "npm install -g typescript vsce"
	}

	stage('Build') {
		env.JAVA_HOME="${tool 'openjdk-11'}"
        env.PATH="${env.JAVA_HOME}/bin:${env.PATH}"
		sh "npm install"
		sh "npm run vscode:prepublish"
	}

	stage('Package') {
		try {
			def packageJson = readJSON file: 'package.json'
			sh "vsce package -o ${PRODUCT_NAME}-vscode-extension-${packageJson.version}-${env.BUILD_NUMBER}.vsix"
			sh "npm pack && mv ${PRODUCT_NAME}-vscode-extension-${packageJson.version}.tgz ${PRODUCT_NAME}-vscode-extension-${packageJson.version}-${env.BUILD_NUMBER}.tgz"
		}
		finally {
			archiveArtifacts artifacts: '*.vsix,**.tgz'
		}
	}
	if(params.UPLOAD_LOCATION) {
		stage('Snapshot') {
			// sh "sftp -C ${UPLOAD_LOCATION}/stable/ <<< \$'mkdir windup-vscode-extension'"
			def filesToPush = findFiles(glob: '**.vsix')
			sh "sftp -C ${UPLOAD_LOCATION}/snapshots/${PRODUCT_NAME}-vscode-extension/ <<< \$'put -p -r ${filesToPush[0].path}'"
			stash name:'vsix', includes:filesToPush[0].path
			def tgzFilesToPush = findFiles(glob: '**.tgz')
			sh "sftp -C ${UPLOAD_LOCATION}/snapshots/${PRODUCT_NAME}-vscode-extension/ <<< \$'put -p -r ${tgzFilesToPush[0].path}'"
			stash name:'tgz', includes:tgzFilesToPush[0].path
		}
	}
}

node('rhel8'){
	if(publishToMarketPlace.equals('true')){
		timeout(time:5, unit:'DAYS') {
			input message:'Approve deployment?', submitter: 'josteele'
		}

		stage("Publish to Marketplace") {
			unstash 'vsix'
			unstash 'tgz'
			withCredentials([[$class: 'StringBinding', credentialsId: 'vscode_java_marketplace', variable: 'TOKEN']]) {
				def vsix = findFiles(glob: '**.vsix')
				sh 'vsce publish -p ${TOKEN} --packagePath' + " ${vsix[0].path}"
			}

            archiveArtifacts artifacts:"**.vsix,**.tgz"

            stage "Promote the build to stable"

            def vsix = findFiles(glob: '**.vsix')
			sh "sftp -C ${UPLOAD_LOCATION}/stable/${PRODUCT_NAME}-vscode-extension/ <<< \$'put -p -r ${vsix[0].path}'"
            def tgz = findFiles(glob: '**.tgz')
			sh "sftp -C ${UPLOAD_LOCATION}/stable/${PRODUCT_NAME}-vscode-extension/ <<< \$'put -p -r ${tgz[0].path}'"

			sh "npm install -g ovsx"
			withCredentials([[$class: 'StringBinding', credentialsId: 'open-vsx-access-token', variable: 'OVSX_TOKEN']]) {
				def packageJson = readJSON file: 'package.json'
				// def vsix = findFiles(glob: '**.vsix')
				sh "ovsx publish -p ${OVSX_TOKEN} ${PRODUCT_NAME}-vscode-extension-${packageJson.version}-${env.BUILD_NUMBER}.vsix"
				// sh 'ovsx publish -p ${OVSX_TOKEN}' + " ${vsix[0].path}"
			}
        }
	}
}

