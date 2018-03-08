# Random AWS cli stuffs
## Delete all my snapshots
    for region in `aws ec2 describe-regions | jq ".[]|.[]|.RegionName"|sed 's/"//g'`
    do
        echo $region
        echo  aws rds --region $region describe-db-snapshots | \
            jq ".[]|.[]|.DBSnapshotArn"
        aws rds --region $region describe-db-snapshots | \
            jq ".[]|.[]|.DBSnapshotArn"
        has_snaps=$(aws rds --region $region describe-db-snapshots | jq ".[]|.[]|.DBSnapshotArn"|wc -l)
        echo "Snaps $has_snaps"
        if [ $has_snaps -ne 0 ]
        then
            aws rds --region $region describe-db-snapshots | \ 
                jq ".[]|.[]|.DBSnapshotIdentifier" | \
                sed 's/"//g' | \
                xargs -L 1 echo aws rds delete-db-snapshot --region $region --db-snapshot-identifier 
            aws rds --region $region describe-db-snapshots | \
                jq ".[]|.[]|.DBSnapshotIdentifier" | \ 
                sed 's/"//g' | \
                xargs -L 1 aws rds delete-db-snapshot --region $region --db-snapshot-identifier 
        fi
        echo -------
    done
