# Week 3 — Decentralized Authentication

Decentralized authentication refers to the process of verifying the identity of a user without relying on a central authority or a single point of control.

### Benefits of Decentralized Authentication
- Enhanced security.
- Privacy.
- Control over personal data. 
- Eliminates the risk of a single point of failure, which can occur in centralized authentication systems.

### Shortcomings of Decentralized Authentication
- Management of decentralized identities.
- Ensuring interoperability between different systems.

### Tools and technologies used for decentralized authentication

- **Blockchain**: Blockchain technology provides a decentralized and tamper-resistant ledger that can be used to store and verify identity information.

- **Decentralized Identity (DID) systems**: DID systems, such as Microsoft's Identity Overlay Network (ION), offer a way to manage and control decentralized identities. These systems enable users to create and manage their own digital identities, which can be used to authenticate their identity.

- **Self-Sovereign Identity (SSI) frameworks**: SSI frameworks, such as Hyperledger Indy, provide a set of standards and protocols for creating and managing decentralized identities. These frameworks enable users to control their identity information and determine who has access to it.

- **Identity wallets**: Identity wallets, such as Metamask, enable users to securely store and manage their decentralized identities and access them when needed.

- **Two-Factor Authentication (2FA) tools**: 2FA tools, such as Google Authenticator or Authy, provide an additional layer of security to decentralized authentication by requiring a second factor, such as a one-time code or biometric authentication, in addition to a password or other credentials.

- **Multi-factor authentication (MFA) tools**: MFA tools, such as Okta or Duo, provide multiple layers of authentication, such as something the user knows (password), something the user has (a smartphone), and something the user is (biometric authentication).

## Amazon Cognito
Amazon Cognito is a managed authentication, authorization, and user management service provided by Amazon Web Services (AWS). It is designed to make it easy for developers to add user sign-up, sign-in, and access control to their web and mobile applications.

### Amazon Cognito's features

- **User sign-up and sign-in**: Amazon Cognito provides a secure and scalable way to enable user registration and authentication using popular identity providers such as Amazon, Facebook, Google, and Apple.

- **User directory management**: Amazon Cognito allows developers to create and manage user directories to store and manage user profiles, groups, and permissions.

- **Security and authentication**: Amazon Cognito provides secure token-based authentication, multi-factor authentication (MFA), and adaptive authentication to protect user accounts from unauthorized access.

- **Integration with AWS services**: Amazon Cognito integrates with other AWS services such as AWS Lambda, Amazon API Gateway, and Amazon S3, making it easy to build scalable and secure applications.

- **Social identity providers**: Amazon Cognito supports popular social identity providers such as Facebook, Google, and Amazon, allowing users to sign in to applications using their existing social media accounts.

- **Customizable user flows**: Amazon Cognito provides customizable user flows for sign-up, sign-in, and password recovery, allowing developers to tailor the user experience to their specific application needs.

## JSON Web Token (JWT)
JSON Web Token (JWT) is a compact and self-contained mechanism for transmitting information between parties as a JSON object. JWTs are often used for authentication and authorization purposes in web applications and APIs.

A JWT consists of three parts: header, a payload and a signature.
- The header contains information about the type of token and the signing algorithm used to generate the signature.
- The payload contains the claims or assertions about the user or entity, such as user ID or role.
- The signature is used to verify the authenticity of the token and ensure that it has not been tampered with.

### Benefits of JWTs

- **Stateless**: JWTs are self-contained and do not require the server to maintain any session state, making them scalable and easier to deploy in distributed systems.

- **Secure**: JWTs can be signed and encrypted to ensure their authenticity and confidentiality.

- **Cross-domain**: JWTs can be used to authorize access to resources across different domains and services.

- **Standardized**: JWTs are a standardized format with well-defined claims and headers, making them interoperable across different systems and platforms.

## Provision Cognito User Group
Using the AWS Console, create a Cognito User Group

## Install AWS Amplify
cd into frontend folder and run
```bash
npm i aws-amplify --save
```

## Configure Amplify
Hook up cognito pool to code in the `App.js`
```js
import { Amplify } from 'aws-amplify';
```

```js
Amplify.configure({
  "AWS_PROJECT_REGION": process.env.REACT_APP_AWS_PROJECT_REGION,
  "aws_cognito_region": process.env.REACT_APP_AWS_COGNITO_REGION,
  "aws_user_pools_id": process.env.REACT_APP_AWS_USER_POOLS_ID,
  "aws_user_pools_web_client_id": process.env.REACT_APP_CLIENT_ID,
  "oauth": {},
  Auth: {
    // We are not using an Identity Pool
    // identityPoolId: process.env.REACT_APP_IDENTITY_POOL_ID, // REQUIRED - Amazon Cognito Identity Pool ID
    region: process.env.REACT_AWS_PROJECT_REGION,           // REQUIRED - Amazon Cognito Region
    userPoolId: process.env.REACT_APP_AWS_USER_POOLS_ID,         // OPTIONAL - Amazon Cognito User Pool ID
    userPoolWebClientId: process.env.REACT_APP_AWS_USER_POOLS_WEB_CLIENT_ID,   // OPTIONAL - Amazon Cognito Web Client ID (26-char alphanumeric string)
  }
});
```

Add the following code in the `docker file` under frontend:
```yml
REACT_APP_AWS_PROJECT_REGION: "${AWS_DEFAULT_REGION}"
REACT_APP_AWS_COGNITO_REGION: "${AWS_DEFAULT_REGION}"
REACT_APP_AWS_USER_POOLS_ID: ""
REACT_APP_CLIENT_ID: ""
```

## Conditionally show components based on logged in or logged out

Inside `HomeFeedPage.js`
```js
import { Auth } from 'aws-amplify';
```

```js
// check if we are authenicated
const checkAuth = async () => {
  Auth.currentAuthenticatedUser({
    // Optional, By default is false. 
    // If set to true, this call will send a 
    // request to Cognito to get the latest user data
    bypassCache: false 
  })
  .then((user) => {
    console.log('user',user);
    return Auth.currentAuthenticatedUser()
  }).then((cognito_user) => {
      setUser({
        display_name: cognito_user.attributes.name,
        handle: cognito_user.attributes.preferred_username
      })
  })
  .catch((err) => console.log(err));
};
```
```js
// check when the page loads if we are authenicated. Already exists so no need to add this.

React.useEffect(()=>{
  loadData();
  checkAuth();
}, [])
```

Update `ProfileInfo.js`

```js
import { Auth } from 'aws-amplify';
```

```js
const signOut = async () => {
  try {
      await Auth.signOut({ global: true });
      window.location.href = "/"
  } catch (error) {
      console.log('error signing out: ', error);
  }
}
```

## Signin Page

```js
import { Auth } from 'aws-amplify';
```

```js
const onsubmit = async (event) => {
    setErrors('')
    event.preventDefault();
    try {
      Auth.signIn(email, password)
        .then(user => {
          localStorage.setItem("access_token", user.signInUserSession.accessToken.jwtToken)
          window.location.href = "/"
        })
        .catch(err => { console.log('Error!', err) });
    } catch (error) {
      if (error.code == 'UserNotConfirmedException') {
        window.location.href = "/confirm"
      }
      setErrors(error.message)
    }
    return false
  }
```

## Signup Page

```js
import { Auth } from 'aws-amplify';
```

```js
  const resend_code = async (event) => {
    setErrors('')
    try {
      await Auth.resendSignUp(email);
      console.log('code resent successfully');
      setCodeSent(true)
    } catch (err) {
      // does not return a code
      // does cognito always return english
      // for this to be an okay match?
      console.log(err)
      if (err.message == 'Username cannot be empty'){
        setErrors("You need to provide an email in order to send Resend Activiation Code")   
      } else if (err.message == "Username/client id combination not found."){
        setErrors("Email is invalid or cannot be found.")   
      }
    }
  }
```

```js
  const onsubmit = async (event) => {
    event.preventDefault();
    setErrors('')
    try {
      await Auth.confirmSignUp(email, code);
      window.location.href = "/"
    } catch (error) {
      setErrors(error.message)
    }
    return false
  }
```

## Recovery Page

```js
import { Auth } from 'aws-amplify';
```

```js
const onsubmit_send_code = async (event) => {
    event.preventDefault();
    setErrors('')
    Auth.forgotPassword(username)
    .then((data) => setFormState('confirm_code') )
    .catch((err) => setCognitoErrors(err.message) );
    return false
  }
```

```js
const onsubmit_confirm_code = async (event) => {
    event.preventDefault();
    setErrors('')
    if (password == passwordAgain){
      Auth.forgotPasswordSubmit(username, code, password)
      .then((data) => setFormState('success'))
      .catch((err) => setErrors(err.message) );
    } else {
      setErrors('Passwords do not match')
    }
    return false
  }
```

```js
  const onsubmit = async (event) => {
    event.preventDefault();
    setErrors('')
    try {
        const { user } = await Auth.signUp({
          username: email,
          password: password,
          attributes: {
              name: name,
              email: email,
              preferred_username: username,
          },
          autoSignIn: { // optional - enables auto sign in after user is confirmed
              enabled: true,
          }
        });
        console.log(user);
        window.location.href = `/confirm?email=${email}`
    } catch (error) {
        console.log(error);
        setErrors(error.message)
    }
    return false
  }
```

## Confirmation Page

```js
import { Auth } from 'aws-amplify';
```

```js
const resend_code = async (event) => {
    setErrors('')
    try {
      await Auth.resendSignUp(email);
      console.log('code resent successfully');
      setCodeSent(true)
    } catch (err) {
      // does not return a code
      // does cognito always return english
      // for this to be an okay match?
      console.log(err)
      if (err.message == 'Username cannot be empty'){
        setErrors("You need to provide an email in order to send Resend Activiation Code")   
      } else if (err.message == "Username/client id combination not found."){
        setErrors("Email is invalid or cannot be found.")   
      }
    }
  }

  const onsubmit = async (event) => {
    event.preventDefault();
    setErrors('')
    try {
      await Auth.confirmSignUp(email, code);
      window.location.href = "/"
    } catch (error) {
      setErrors(error.message)
    }
    return false
  }
```

## Authenticating Server Side

Add in the `HomeFeedPage.js` a header to pass along the access token

```js
  headers: {
    Authorization: `Bearer ${localStorage.getItem("access_token")}`
  }
  ```

In the `app.py`

```py
cors = CORS(
  app, 
  resources={r"/api/*": {"origins": origins}},
  headers=['Content-Type', 'Authorization'], 
  expose_headers='Authorization',
  methods="OPTIONS,GET,HEAD,POST"
)
```