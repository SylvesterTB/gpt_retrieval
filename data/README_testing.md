# How to test with the my.KUKA test user

## ID Token retrieval
1. Log into the AWS dev account (https://kuka.awsapps.com/start#/)
1. Go to Cognito (https://eu-central-1.console.aws.amazon.com/cognito/home?region=eu-central-1#) and click "Manage User Pools"
1. Select the UpdateServiceUserPool
1. Select "App integration - App client settings"
1. Scroll to the end and click "Launch Hosted UI"
    * Note: Currently, the link is https://rus-dev-updateservices.auth.eu-central-1.amazoncognito.com/login?client_id=146hlov43p9h4d52crhii7k7ov&response_type=token&scope=email+openid+profile&redirect_uri=http://localhost:3000/callback, but this will change with a redeployment of the Cognito User Pool.
1. Click "Login-with-myKUKA"
1. Use the following credentials
    * Username: tobmail30+update0815@gmail.com
    * Password: Ask someone from the team
1. You will be redirected to a localhost port which obviously fails.
* **Copy the URL to a text editor and extract the ID token, it is passed as a parameter (...&id_token=...)**

## Adjust test cases
Set the variable `idToken` in backend_test.go to the ID token you retrieved as described above.

Note: If you access the backend via a test and it once failed because the session was not valid, remember to clear the test cache after you have renewed your session, so that go actually re-runs the test:
`go clean -testcache`

Note #2: In case of api error : `RequestTimeTooSkewed: The difference between the request time and the current time is too large.` do `vagrant halt`, `vagrant up` and clean testcache.

## AWS CLI
You can also use the AWS CLI after retrieving the ID token. For this, you need to manually execute the steps that are performed by backend.go as described below (replace placeholders in «» with their respective values):

1. Retrieve the identity id:

    ```aws cognito-identity get-id --identity-pool-id eu-central-1:0b0d312b-1cb2-4e77-bbcb-b7943e074cf6 --logins cognito-idp.eu-central-1.amazonaws.com/eu-central-1_MvdeK128H=«ID_TOKEN»```

1. Retrieve the credentials:

    ```aws cognito-identity get-credentials-for-identity --identity-id «IDENTITY_ID» --logins cognito-idp.eu-central-1.amazonaws.com/eu-central-1_MvdeK128H=«ID_TOKEN»```

1. Copy the values you retrieved to `~/.aws/credentials`:

    ```
    [default]
    aws_access_key_id = ...
    aws_secret_access_key = ...
    aws_session_token = ...
    ```