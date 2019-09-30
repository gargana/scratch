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
## Delete log groups
```bash
for i in $(aws logs describe-log-groups | jq -r ".|.[]|.[]|.logGroupName")
do
  echo "Delete '$i'"
  aws logs delete-log-group --log
-group-name $i
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
