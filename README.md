# tfe-mounted-to-external
This is a test repository to learn how to have a successful migration from TFE mounted disk installation to External Services.

The following KB article has been followed to have a successful migration: 
[Migrate TFE from Mounted Disk to External Servia via the Backup/Restore API](https://support.hashicorp.com/hc/en-us/articles/10536697730323-Migrate-TFE-from-Mounted-Disk-to-External-Services-mode-with-Backup-Restore-API)


### Prerequisites

- [X] [Terraform](https://www.terraform.io/downloads)
- [X] [TFE Mounted Disk installation up and running](https://github.com/dlavric/tfe-aws-auto)
- [X] [New and empty TFE External Services installation](https://github.com/dlavric/tfe-active-active-aws)

## How to Use this Repo

- Clone this repository:
```shell
git clone git@github.com:dlavric/tfe-mounted-to-external.git
```

- Go to the directory where the repo is stored:
```shell
cd tfe-mounted-to-external
```

- Connect to the running TFE instance with a mounted disk installation with your `.pem` file
```shell
ssh -i "daniela-key.pem" ubuntu@ec2-35-89-160-155.us-west-2.compute.amazonaws.com
```

- Check the TFE version that is running on this instance. take note of the `sequence` value
```shell
replicatedctl app status

[
    {
        "AppID": "8311d2251e98455a5e650977bee65331",
        "Sequence": 703,
        "PatchSequence": 0,
        "State": "started",
        "DesiredState": "started",
        "Error": "",
        "IsCancellable": false,
        "IsTransitioning": false,
        "LastModifiedAt": "2023-06-28T09:01:49.500895652Z"
    }
]
```

- Create a new single TFE installation with External Services with no organization, no initial user, using this repository [New and empty TFE External Services installation](https://github.com/dlavric/tfe-active-active-aws) by commenting L192-212 of this [script](https://github.com/dlavric/tfe-active-active-aws/blob/main/tf-script/user-data-single.sh#L169-L213)


- Now, connect to the new TFE instance with External Services 
```shell
ssh -J ubuntu@daniela-tfe-client.tf-support.hashicorpdemo.com ubuntu@<internal ip address of the TFE server>
```

- Make sure this instance is running the same TFE version
```shell
replicatedctl app status
```

- Now take note of the `backup` token from your TFE settings, from both instances: mounted disk and external services and save them
```shell
replicatedctl app-config export --hidden | jq -r '.backup_token.value'

exxxxxxxx0
```

- Create a Backup using the [Backup API](https://developer.hashicorp.com/terraform/enterprise/admin/infrastructure/backup-restore#creating-a-backup)

- Create the payload file named `payload.json` with the following content on the Source TFE (Mounted Disk)
```shell
vim payload.json

{
   "password": "backup-password"
}
```

- Export the backup token as an environment variable
```shell
export TOKEN=<source-backup-token>
```

- Execute the following API request, for your TFE hostname
```shell
curl -k \
  --header "Authorization: Bearer $TOKEN" \
  --request POST \
  --data @payload.json \
  --output backup.blob \
  https://34.208.218.105.nip.io/_backup/api/v1/backup
```

- Check the backup has been taken and it should look like this
```shell
file backup.blob

backup.blob: data
``` 

- Stop the TFE SOURCE application so no other changes are done on this instance
```shell
replicatedctl app stop
```

- Connect to the TFE External Services installation
```shell
ssh -J ubuntu@daniela-tfe-client.tf-support.hashicorpdemo.com ubuntu@<internal ip address of the TFE server>
```

- Note the backup token of this destination instance
```shell
replicatedctl app-config export --hidden | jq -r '.backup_token.value'
```

- Create the same payload file on the server
```shell
vim payload.json

{
   "password": "backup-password"
}
```

- Export the backup token as an environment variable
```shell
export TOKEN=<destination-backup-token>
```


- Restore the backup
```shell
curl -k \
  --header "Authorization: Bearer $TOKEN" \
  --request POST \
  --form config=@payload.json \
  --form snapshot=@backup.blob \
  https://10.234.11.241/_backup/api/v1/restore
```

- Restart the TFE instance
```shell
replicatedctl app stop

replicatedctl app status

replicatedctl app start
```
