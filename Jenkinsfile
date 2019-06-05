#!/usr/bin/env groovy

// PARAMETERS for this pipeline:
// branchToBuildCTL = refs/tags/20190401211444 or master
// DESTINATION = user@host_or_ip:/path/to/che-incubator/chectl
// BASE_URL = http://host_or_ip/path/to/che-incubator/chectl

def installNPM(){
	def nodeHome = tool 'nodejs-10.15.3'
	env.PATH="${env.PATH}:${nodeHome}/bin"
	sh "npm install -g yarn pkg cpx rimraf"
	sh "npm version"
}

// TODO: re-add win-x64; fails due to missing 7zip: "Error: install 7-zip to package windows tarball"
def platforms = "linux-x64,darwin-x64,linux-arm"
def CTL_path = "chectl"
//def VER_CTL = "VER_CTL"
def SHA_CTL = "SHA_CTL"
timeout(180) {
	node("rhel7-releng"){ stage "Build ${CTL_path}"
		cleanWs()
		checkout([$class: 'GitSCM', 
			branches: [[name: "${branchToBuildCTL}"]], 
			doGenerateSubmoduleConfigurations: false, 
			poll: true,
			extensions: [[$class: 'RelativeTargetDirectory', relativeTargetDir: "${CTL_path}"]], 
			submoduleCfg: [], 
			userRemoteConfigs: [[url: "https://github.com/che-incubator/${CTL_path}.git"]]])
		installNPM()
		SHA_CTL = sh(returnStdout:true,script:"cd ${CTL_path}/ && git rev-parse --short=4 HEAD").trim()
		sh "cd ${CTL_path}/ && sed -i -e 's#version\": \"\\(.*\\)\",#version\": \"\\1-'${SHA_CTL}'\",#' package.json"
		sh "cd ${CTL_path}/ && grep -v oclif package.json | grep -e version"

		try {
			sh "cd ${CTL_path}/ && npm install --verbose && yarn pack --verbose"
		}
		catch (Exception e) {
			echo "[ERROR] npm failed: ${e}"
			archiveArtifacts fingerprint: false, artifacts:"/home/hudson/.npm/_logs/*-debug.log, **/*.log, **/*logs/**, **/*.tar.gz"
		}

		// TODO remove this in favour of oclif
		//sh "cd ${CTL_path}/ && pkg . -t node10-linux-x64,node10-macos-x64,node10-win-x64 --options max_old_space_size=1024 --out-path ./bin/ && ls -la ./bin/"
		//stash name: 'stashBin', includes: findFiles(glob: "${CTL_path}/bin/chectl-*").join(", ")

		sh "cd ${CTL_path}/ && npx oclif-dev pack -t ${platforms} && find ./dist/ -name \"*.tar*\""
		stash name: 'stashDist', includes: findFiles(glob: "${CTL_path}/dist/").join(", ")
	}
}
timeout(180) {
	node("rhel7-releng"){ stage "Publish ${CTL_path}"
		//unstash 'stashBin'
		//sh "cd ${CTL_path}/ && ls -laR ./bin/ && rsync -arzq --protocol=28 ./bin/chectl-* --delete ${DESTINATION}/bin/"

		unstash 'stashDist'
		sh '''#!/bin/bash -xe
		cd ${CTL_path}/ && find ./dist/ -name \"*.tar*\" && rsync -arzq --protocol=28 ./dist/channels/${SHA_CTL}/* ${DESTINATION}/dist/

		for platform in ${platforms//,/ }; do
			cat << EOF > /tmp/${platform}
{
"version" : "${SHA_CTL}",
"channel": "stable",
"gz" : "${BASE_URL}/dist/chectl-v${SHA_CTL}/chectl-v${SHA_CTL}-${platform}.tar.gz"
}
EOF
		done
		'''

		archiveArtifacts fingerprint: false, artifacts:"**/*.log, **/*logs/**, **/*.tar.gz"

	}
}