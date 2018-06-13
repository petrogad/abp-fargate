# abp-fargate

[aws-blueprint](https://github.com/rynop/aws-blueprint) aws-blueprint example for a ECS fargate based app, with or without an ELB

## Setup

1. Create an ECS [image repository](https://console.aws.amazon.com/ecs/home?region=us-east-1#/repositories).  Naming convention `<app>/<branch>`. Populate it with an inital image. See [examples/fargate/Dockerfile](examples/fargate/Dockerfile) for an example:
    1. `aws ecr get-login --no-include-email --region us-east-1`
    1. `docker build -t ecs-test/master:initial .`
    1. `docker tag ecs-test/master:initial 1111.dkr.ecr.us-east-1.amazonaws.com/ecs-test/master:initial`
    1. `docker push 1111.dkr.ecr.us-east-1.amazonaws.com/ecs-test/master:initial`
1. Use [vpc-ecs-cluster-resources.yaml](./aws/cloudformation/vpc-ecs-cluster-resources.yaml) to create an ECS Cluster inside its own VPC. This cluster will run all your stages (task per stage). CloudFormation stack naming convention: `<project>--ecs-cluster`
1. Setup your environment variables in [Systems manager parameter store](https://console.aws.amazon.com/systems-manager/parameters).  We recommend using the namespace convention `/<stage>/<repoName>/<branch>/<app>/ecsEnvs/<env var name>`. [aws-env](https://github.com/Droplr/aws-env) is used to load the env vars into your container.  Ex: `/prod/abp-fargate/master/ResizeImage/ecsEnvs/MY_KEY`
1.  Create a Github user (acct will just be used to read repos for CI/CD), give it read auth to your github repo.  Create a personal access token for this user at https://github.com/settings/tokens.  This token will be used by the CI/CD to pull code.
1.  Review **Code Specifics** below
1. Use [cloudformation-test-staging-prod.yaml](/pipelines/cicd/cloudformation-test-staging-prod.yaml) to create a codepipeline that builds docker a image and updates ECS with stage promotion approval. CloudFormation stack naming convention: `<app>--<branch>--<service>`.  The pipeline will create a CloudFormation stack for each stage.  Stage specific parameters are set in [./aws/cloudformation/parameters](./aws/cloudformation/parameters/).
    1. For param `RelCloudFormationTemplatePath`: if your app needs an ELB specify [aws/cloudformation/fargate-with-elb.yaml](./aws/cloudformation/fargate-with-elb.yaml) otherwise, specify [aws/cloudformation/fargate-no-elb.yaml](./aws/cloudformation/fargate-no-elb.yaml). 
    1. If using `fargate-with-elb.yaml` your app **MUST**:
        * accept health checks at `/healthcheck`
        * Verify the value of the `X-From-CDN` header matches the value you set in the `VerifyFromCfHeaderVal` parameter in `<stage>--ecs-codepipeline-parameters.json` 
        * Edit your cloudfront > dist settings > change Security policy to `TLSv1.1_2016`.  CloudFormation does not support this parameter yet.
        * Create a DNS entry in route53 for production that consumers will use.  The cloud formation creates one for `prod--` but you do not want to use this as the CloudFormation can be deleted.

### Code specifics

This example is using golang and the [Twirp RPC framework](https://github.com/twitchtv/twirp).  Project layout is based on [golang-standards/project-layout](https://github.com/golang-standards/project-layout)

We recommend using [retool](https://github.com/twitchtv/retool) to manage your tools (like (dep)[https://github.com/golang/dep]).  Why?  If you work with anyone else on your project, and they have different versions of their tools, everything turns to shit.

1.  Update [Dockerfile](./build/Dockerfile) to set your github repo.
1.  Update the code to use your go package, by doing an extended file and replace of all occurances of `rynop/abp-fargate` with your golang package namespace.
1. (Install retool)[https://github.com/twitchtv/retool#usage]: `go get github.com/twitchtv/retool`. Make sure to add `$GOPATH/bin` to your PATH
1. These commands should be run in your go projects
    1.  `retool add github.com/golang/dep/cmd/dep origin/master`
    1.  `retool add github.com/golang/lint/golint origin/master`
    1.  `retool add github.com/golang/protobuf/protoc-gen-go origin/master`
    1.  `retool add github.com/twitchtv/twirp/protoc-gen-twirp origin/v6_prerelease`    
    1.  `retool do dep init`.  In this case, since we already have a `dep` project setup, run `retool do dep ensure`
1. Add dependency example: `retool do dep ensure -add github.com/apex/gateway github.com/aws/aws-lambda-go`
1.  Auto-generate the code:
```
retool do protoc --proto_path=$GOPATH/src:. --twirp_out=. --go_out=. ./rpc/publicservices/service.proto 
retool do protoc --proto_path=$GOPATH/src:. --twirp_out=. --go_out=. ./rpc/adminservices/service.proto 
```    
1. For this example, the interface implementations have been hand created in `pkg/`. Take a look.
1. Example to consume twirp API in this example: `curl -H 'Content-Type:application/json' -H 'Authorization: Bearer aaa' -H 'X-FROM-CDN: <your VerifyFromCfHeaderVal>' -d '{"term":"wahooo"}' https://<--r output CNAME>/com.rynop.twirpl.publicservices.Image/CreateGiphy`

Testing locally:
1.  Set `LOCAL_LISTEN_PORT` and `X_FROM_CDN` env vars. (Fish: `set -gx LOCAL_LISTEN_PORT 8080`, `set -gx X_FROM_CDN localTest`)
1.  Build & run: `cd cmd/example-webservices`, `go build -o /tmp/main .; /tmp/main`
1.  Hit endpoint: `curl -v -H 'Content-Type:application/json' -H 'Authorization: Bearer aaa' -H 'X-FROM-CDN: localTest' -d '{"term":"wahooo"}' http://localhost:8080/com.rynop.twirpl.publicservices.Image/CreateGiphy`


## Building docker image locally

From your project room run: `docker build -f build/Dockerfile -t <repo>/<branch>:<tag>`
