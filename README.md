# Exploration of AWS-CDK
Even though cdk can be written in several languages it seems the easiest is
typescript. Partly this is because cdk is typescript natively and has the
greatest number of examples. Also The documentation in non typescript languages
is wretched. So write cdk in typescript unless you're a masochist.

## Learning CDK

* [Main Repo](https://github.com/awslabs/aws-cdk)
* [Samples](https://github.com/aws-samples/aws-cdk-examples)
* [Docs](https://docs.aws.amazon.com/CDK/latest/userguide/what-is.html)
* [Demo from re:invent 2018](https://github.com/awslabs/cdk-reinvent)
* [Deep dive into AWS Cloud Development Kit](https://www.youtube.com/watch?v=9As_ZIjUGmY)
* [Another nice tutorial](https://cdkworkshop.com/)
* [slack channel](https://gitter.im/awslabs/aws-cdk)

## Helpful hints

### cdk project with codepipeline/codebuild.

cdk expects env vars to exist. environments are mapped to git branches:

* master -> prod environment
* !master -> !master environment

This wretched hack captures the branch that triggers a codepipeline source phase for the ensuing codebuild phase.

```
#!/bin/bash
if [ -z ${CODEBUILD_INITIATOR+x} ];
then
echo "CODEBUILD_INITIATOR is unset";
else
. scripts/codebuild/env.sh;
echo "CODEBUILD_INITIATOR is set to '$CODEBUILD_INITIATOR'";
#horrible hack to set git branch env var
pipeline_name=$(echo $CODEBUILD_INITIATOR |awk -F '/' '{print $2}')
echo  "pipeline name is  $pipeline_name"
pipeline_deets=$(aws codepipeline get-pipeline --name $pipeline_name)
echo  "pipeline details"
echo $pipeline_deets | jq .
export CODEPIPELINE_GIT_BRANCH_NAME=$(echo $pipeline_deets | jq -r '.pipeline.stages[] | select (.name=="Source") .actions[].configuration.BranchName')
echo  "git branch name is  $CODEPIPELINE_GIT_BRANCH_NAME"
fi
```

# How to simulate codebuild
You can reverse engineer buildspec.yaml if you want to manually run things like "cdk synth", "cdk diff", "cdk deploy", etc. But wouldn't you rather just run codebuild locally to test your infrastructure as code? Here's how:

* clone code to build local codebuild docker image in  ~/environment

```
mkdir -p ~/github/aws/ /tmp/artifacts
git clone https://github.com/aws/aws-codebuild-docker-images.git ~/github/aws/
```

* build local docker image to run codebuild, this will take some time and take ~10GB disk space.
see hints down below for how to embiggen a c9 instance for more disk space.

```
cd ~/github/aws/aws-codebuild-docker-images/ubuntu/standard/4.0/
docker build -t aws/codebuild/standard:4.0 .
```

* set region so docker runtime will inherit with -c

```
export AWS_REGION="us-west-2"
```

* run codebuild locally

```
~/github/aws/aws-codebuild-docker-images/local_builds/codebuild_build.sh -i aws/codebuild/standard:4.0 -a /tmp/artifacts -s ~/your/app/repo/with/buildspec/at/root -c
```

Boom!


## Useful cdk ops

 * `npm run build`   compile typescript to js
 * `npm run watch`   watch for changes and compile
 * `cdk synth`       emits the synthesized CloudFormation template, also creates synthesized stack templates in cdk.out
 * `for i in cdk.out/*Stack.template.json; do cfn_nag_scan -i $i; done`   scan synthesized stack templates in cdk.out
 * `cdk deploy --require-approval never -t Application=demo -t ProductOwner=eric.odell@ucop.edu -t Environment=dev`      deploy this stack to your default AWS account/region
 * `cdk diff`        compare deployed stack with current state
 * ` for i in `grep peer /tmp/i | awk '{print $8}' |sort | uniq |awk -F ^ '{print $1"latest"}'`; do npm install --save-dev $i; done`  capture output of npm i with "npm WARN @aws-cdk/aws-stepfunctions@1.11.0 requires a peer of @aws-cdk/aws-iam@^1.11.0 but none is installed. You must install peer dependencies yourself."

# npm

## updates

Hint use this to make package updates easier

```
npm install -g npm-check-updates
ncu -u && npm i #update package.json and update
ncu -g # show updates to global packages
```

## scripts

scripts key in package.json contains various scripts (duh!) that can perform useful tasks.

```
cat package.json  | jq .scripts
```

## testing with jest

```
npm run test
```

## doc with [typedoc](https://typedoc.org/guides/doccomments/)

```
npm run doc
```

## linting with tslint

```
npm run lint
```

## formatting with prettier

```
npm run format
```

## scanning with cfn_nag_scan

```
npm run format
```


# cloud9

* notes on how to embiggen c9 instance

```
# Get the ID of the envrionment host Amazon EC2 instance.

INSTANCEID=$(curl http://169.254.169.254/latest/meta-data//instance-id)

# Get the ID of the Amazon EBS volume associated with the instance.

VOLUMEID=$(aws ec2 describe-instances --instance-id $INSTANCEID | jq -r .Reservations[0].Instances[0].BlockDeviceMappings[0].Ebs.VolumeId)

# grow volume by how ever much you need, here's an example of changing to 40 GB

aws ec2 modify-volume --volume-id  $VOLUMEID --size 40
aws ec2 describe-volumes --volume-ids $VOLUMEID

# figure out which disk you want to embiggen, here's an example using ubuntu c9 instance

sudo growpart /dev/nvme0n1 1
sudo resize2fs /dev/nvme0n1p1
```
