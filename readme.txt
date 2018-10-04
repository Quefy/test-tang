#!groovy 
/*
    コマンドプロンプトを「管理者として実行」で開き、
    git config --system --unset credential.helperなどを実行して資格情報ヘルパーを使用しないように変更
*/
def git_init(String repoUrl){
    def ret = 0

    withCredentials([usernamePassword(credentialsId:"GitHub",usernameVariable:"username",passwordVariable:"password")]){
        ret = bat returnStatus:true,script:"git branch"
        echo "${ret}"
        if( ret != 0 ){
            bat """
                git init
                git config http.proxy http://tci-proxy.trans-cosmos.co.jp:8080
                git config https.proxy http://tci-proxy.trans-cosmos.co.jp:8080
                git remote add origin ${repoUrl}
                git fetch origin
                git branch -a
            """
            return true
        }
        return false
    }
}

Boolean git_check_updated(String branchName){
    String retStdout = ""
    Integer retStatus = 0

    withCredentials([usernamePassword(credentialsId:"GitHub",usernameVariable:"username",passwordVariable:"password")]){
        bat "git checkout ${branchName}"
        bat "git fetch origin"
        retStdout = bat returnStdout:true,script:"@git diff origin/${branchName} --name-only"
        echo "retStdout=${retStdout.trim()}"
    }

    if(retStdout.trim().length() == 0 ){
        echo "no updated"
        return false
    }

    return true
}

def git_push(String lBranchName,String rBranchName){
    withCredentials([usernamePassword(credentialsId:"GitHub",usernameVariable:"username",passwordVariable:"password")]){
        bat "git push -f origin ${lBranchName}:${rBranchName}"
    }
}

def git_pull(String branchName){
    withCredentials([usernamePassword(credentialsId:"GitHub",usernameVariable:"username",passwordVariable:"password")]){    
        bat "git pull origin ${branchName}"
    }
}

stage 'test'
node{
    Boolean isFirstInit = false
    Boolean isUpdated = false    

    isFirstInit = git_init("https://github.com/Quefy/test-tang.git")
    isUpdated = git_check_updated("resolve_wrk")

    if(isFirstInit == true || isUpdated == true){
        if(isUpdated == true){
            git_pull("resolve_wrk")
        }
        git_push("resolve_wrk","resolve")
    }
}
