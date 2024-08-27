# How to setup the AWS profile
- Login the AWS console. The use must have the required privileges
- At the right top, click your username to open the drop down menu and select 'security credentials' to open the credentials management.
- search 'Access keys' section and click the button 'create access key'
- Follow the process to get the access key id and the access key.
- Edit ~/.aws/credentials with a file editor to add the profile
```
[oss]
aws_access_key_id = ***
aws_secret_access_key = ***
```

