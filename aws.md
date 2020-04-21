# Random AWS cli stuff:
## Delete all my RDS Snapshots:
Quickly iterate over my account and find snapshots and delete them
```bash
for region in $(aws ec2 describe-regions | jq -r ".[]|.[]|.RegionName")
do
    echo $region
    echo 'aws rds --region $region describe-db-snapshots | \
        jq ".[]|.[]|.DBSnapshotArn"' 
    aws rds --region $region describe-db-snapshots | \
        jq ".[]|.[]|.DBSnapshotArn"
    has_snaps=$(aws rds --region $region describe-db-snapshots | jq ".[]|.[]|.DBSnapshotArn"|wc -l)
    echo "Snaps $has_snaps"
    if [ $has_snaps -ne 0 ]
    then
        aws rds --region $region describe-db-snapshots | jq ".[]|.[]|.DBSnapshotIdentifier" | sed 's/"//g' | xargs -L 1 echo aws rds delete-db-snapshot --region $region --db-snapshot-identifier 
        aws rds --region $region describe-db-snapshots | jq ".[]|.[]|.DBSnapshotIdentifier" | sed 's/"//g' | xargs -L 1 aws rds delete-db-snapshot --region $region --db-snapshot-identifier

    fi

    has_options=$(aws rds --region $region describe-option-groups | jq ".[]|.[]|select(.OptionGroupName|test(\"^(?!default).\"))|.OptionGroupArn"|wc -l)
    echo "Options $has_options"
    if [ $has_options -ne 0 ]
    then
        aws rds --region $region describe-option-groups | jq ".[]|.[]|select(.OptionGroupName|test(\"^(?!default).\"))|.OptionGroupName" | sed 's/"//g' | xargs -L 1 echo aws rds delete-option-group --region $region --option-group-name
        aws rds --region $region describe-option-groups | jq ".[]|.[]|select(.OptionGroupName|test(\"^(?!default).\"))|.OptionGroupName" | sed 's/"//g' | xargs -L 1 aws rds delete-option-group --region $region --option-group-name
    fi

    echo -------
done
```

## Delete all TaskCat stacks
```bash
for i in $(aws ec2 describe-regions|jq -r ".[]|.[]|.RegionName")
do
  echo $i
  for STACK in $(aws cloudformation describe-stacks --region $i | jq -r ".[]|.[]|.StackName")
  do
    echo $STACK
    echo ${STACK:0:4}
    if [ "${STACK:0:4}" = "tCaT" ]
    then 
      echo tCatFound
      aws cloudformation delete-stack --stack-name "$STACK" --region $i
    fi
  done
done
```

## Search CloudTrail for stack launch failures at the CreateStack API level
```bash
DAY_IN_SECONDS=$(echo 60*60*24 | bc)
DAYS_BACK=7
START_TIME=$(echo "$(date +%s)-($DAYS_BACK*$DAY_IN_SECONDS)" | bc)
END_TIME=$(echo $START_TIME+$DAY_IN_SECONDS | bc)
# Display start/end times in human parseable format
  
echo -n "Start: "
date --date="@$START_TIME"
echo -n "End: "
date --date="@$END_TIME"

for REGION in $(aws ec2 describe-regions | jq ".[]|.[]|.RegionName"|sed 's/"//g')
do
  echo $REGION
  
  # TODO: Looking a day at a time. This may be too much info so need to make more granular
  aws cloudtrail lookup-events \
    --region $REGION \
    --lookup-attributes AttributeKey=EventName,AttributeValue=CreateStack \
    --start-time $START_TIME \
    --end-time $END_TIME \
    jq -r ".[]|.[]|.CloudTrailEvent" | \
    jq "select(.errorCode != null)|.errorCode, .errorMessage"
done
```
## Generate AMI List for CFN (see here for automatically handled codes: https://github.com/aws-quickstart/taskcat/blob/master/taskcat/cfg/amiupdater.cfg.yml)
```bash
#!/bin/sh

AMI_CODE="CENTOS7"
AMI_DESCRIPTION="CentOS 7 (x86_64) - with Updates HVM"
AMI_FILTER="description"
AMI_FILTER_VALUE="CentOS Linux 7 x86_64 HVM EBS ENA 19*"
AMI_OWNER_ID="679593333241"

AMI_CODE="CISUBUNTU1604"
AMI_DESCRIPTION="CIS 1 Ubuntu 16.04"
AMI_FILTER="product-code"
AMI_FILTER_VALUE="2l0khimiqztu90zd64xu99tz5"
AMI_OWNER_ID="679593333241"

AMI2_CODE="US1604HVM"
AMI2_DESCRIPTION="Ubuntu 16.04 LTS"
AMI2_FILTER="name"
AMI2_FILTER_VALUE="ubuntu/images/hvm-ssd/ubuntu-xenial-16.04-amd64-server-????????*"
AMI2_OWNER_ID="099720109477"

echo "Metadata:"
echo "  AWSAMIRegionMap:"
echo "    Filters:"
echo "      ${AMI_CODE}:"
echo "        ${AMI_FILTER}: 2l0khimiqztu90zd64xu99tz5"
echo "Mappings:"
echo "  AWSAMIRegionMap:"
echo "    AMI:"
echo "      ${AMI_CODE}: \"${AMI_DESCRIPTION}\""
echo "      ${AMI2_CODE}: \"${AMI2_DESCRIPTION}\""


for reg in $(aws ec2 describe-regions | jq -r ".|.Regions[]|.RegionName")
do
  echo "    $reg:"
  AMI_ID=$(aws ec2 describe-images --region $reg --filters "Name=${AMI_FILTER},Values='${AMI_FILTER_VALUE}'" "Name=owner-id,Values=${AMI_OWNER_ID}" | \
          jq -r -s ".|.[]|.[]|sort_by(.CreationDate)|.[-1]|.ImageId")
  DESC=$(aws ec2 describe-images --region $reg --filters "Name=${AMI_FILTER},Values='${AMI_FILTER_VALUE}'" "Name=owner-id,Values=${AMI_OWNER_ID}" | \
          jq -r -s ".|.[]|.[]|sort_by(.CreationDate)|.[-1]|.Name")
  AMI2_ID=$(aws ec2 describe-images --region $reg --filters "Name=${AMI2_FILTER},Values='${AMI2_FILTER_VALUE}'" "Name=owner-id,Values=${AMI2_OWNER_ID}" | \
          jq -r -s ".|.[]|.[]|sort_by(.CreationDate)|.[-1]|.ImageId")
  DESC2=$(aws ec2 describe-images --region $reg --filters "Name=${AMI2_FILTER},Values='${AMI2_FILTER_VALUE}'" "Name=owner-id,Values=${AMI2_OWNER_ID}" | \
          jq -r -s ".|.[]|.[]|sort_by(.CreationDate)|.[-1]|.Name")
  # aws ec2 describe-images --region $reg --filters "Name=description,Values='CentOS Linux 7*'" "Name=owner-id,Values=679593333241" | \
  #jq -s ".|.[]|.[]|sort_by(.CreationDate)|.[-1]|.Name,.CreationDate,.Description,.ImageId,.OwnerId"
  echo "      ${AMI_CODE}: ${AMI_ID} #${DESC}"
  echo "      ${AMI2_CODE}: ${AMI2_ID} #${DESC2}"
done
```
## Generate uniform set of SSH keys for taskcat testing:
```bash
KEYPAIR_NAME=somekeynamehere
for i in $(aws ec2 describe-regions|jq -r ".[]|.[]|.RegionName")
do
    echo $i
    aws ec2 delete-key-pair --region $i --key-name=$KEYPAIR_NAME 2>&1 > /dev/null
    KEY_JSON=$(aws ec2 create-key-pair --region $i --key-name $KEYPAIR_NAME)
    echo $KEY_JSON | jq -r ".KeyMaterial" | tr "\\n" "\n" > ~/.ssh/${KEYPAIR_NAME}-${i}
    chmod 600 ~/.ssh/${KEYPAIR_NAME}-${i}
done

# results in:
# ~/.ssh/${KEYPAIR_NAME}-${REGION}
# eg:
# ~/.ssh/somekeynamehere-us-east-1
# ~/.ssh/somekeynamehere-us-east-2
# ~/.ssh/somekeynamehere-us-west-1
# ~/.ssh/somekeynamehere-us-west-2
# ...
# ...
```

## Cleanup LogGroups:
```bash
PATTERN=AWXStack

#for region in $(aws ec2 describe-regions | jq -r ".Regions[]|.RegionName")
#do
#    echo $region:
#    for i in $(aws logs describe-log-groups --region ${region}| jq -r ".|.[]|.[]|.logGroupName")
#    do
#        echo "lg: '$i'"
#    done
#done

for region in $(aws ec2 describe-regions | jq -r ".Regions[]|.RegionName")
do
    echo $region:
    for i in $(aws logs describe-log-groups --region ${region} | jq -r ".|.[]|.[]|select(.logGroupName | contains(\"${PATTERN}\"))|.logGroupName")
    do
        echo "Delete '$i'"
        aws logs delete-log-group --region ${region} --log-group-name $i
    done
done
```
