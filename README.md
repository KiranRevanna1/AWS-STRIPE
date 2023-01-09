# Working with secrets
## Setup a Stripe Account
We aren’t going to do much else in the way of storing this info in our database. We’ll leave that as an exercise for the reader.

### Sign up for Stripe
Let’s start by creating a free Stripe account. Head over to Stripe and register for an account.

<img src="https://imgur.com/Ft1YQ3j.png" alt="Homepage view 3" width=1000 height=500/>
Once signed in, click the Developers link on the left.
<img src="https://imgur.com/j0JsEHr.png" alt="Homepage view 3" width=1000 height=500/>
And hit API keys.
<img src="https://imgur.com/ySDXEHO.png" alt="Homepage view 3" width=1000 height=500/>
The first thing to note here is that we are working with a test version of API keys. To create the live version, you’d need to verify your email address and business details to activate your account. For the purpose of this guide we’ll continue working with our test version.

The second thing to note is that we need to generate the <b>Publishable key</b> and the <b>Secret key<b>. The Publishable key is what we are going to use in our frontend client with the Stripe SDK. And the Secret key is what we are going to use in our API when asking Stripe to charge our user. As denoted, the Publishable key is public while the Secret key needs to stay private.

Hit the <b>Reveal test key token.<b>
<img src="https://imgur.com/ZDC7Kwr.png" alt="Homepage view 3" width=1000 height=500/>
Make a note of both the Publishable key and the Secret key. We are going to be using these later.

Next, let’s use this in our SST app.

## Handling Secrets in SST
We are going to create a `.env` file to store this.

 Create a new file in `.env.local` with the following.
```sh
STRIPE_SECRET_KEY=STRIPE_TEST_SECRET_KEY
```
Make sure to replace the `STRIPE_TEST_SECRET_KEY` with the Secret key from the previous chapter.

SST automatically loads this into your application.

A note on committing these files. SST follows the convention used by Create React App and others of committing .env files to Git but not the `.env.local` or `.env.$STAGE.local` files. You can read more about it here.

To ensure that this file doesn’t get committed, we’ll need to add it to the `.gitignore` in our project root. You’ll notice that the starter project we are using already has this in the `.gitignore`.
```sh
# environments
.env*.local
```
Also, since we won’t be committing this file to Git, we’ll need to add this to our CI when we want to automate our deployments. We’ll do this later in the guide.

Next, let’s add these to our functions.

 Add the following below the `TABLE_NAME: table.tableName`, line in `stacks/ApiStack.js`:
```sh
STRIPE_SECRET_KEY: process.env.STRIPE_SECRET_KEY,
```
We are taking the environment variables in our SST app and passing it into our API.

## Add a Billing Lambda
 Start by installing the Stripe NPM package. Run the following in the `services/` folder of our project.
```sh
$ npm install stripe
```
 Create a new file in `services/functions/billing.js` with the following.
```sh
import Stripe from "stripe";
import handler from "../util/handler";
import { calculateCost } from "../util/cost";

export const main = handler(async (event) => {
  const { storage, source } = JSON.parse(event.body);
  const amount = calculateCost(storage);
  const description = "Scratch charge";

  // Load our secret key from the  environment variables
  const stripe = new Stripe(process.env.STRIPE_SECRET_KEY);

  await stripe.charges.create({
    source,
    amount,
    description,
    currency: "usd",
  });

  return { status: true };
});
```
Most of this is fairly straightforward but let’s go over it quickly:
<ul>
<li>We get the `storage` and `source` from the request body. The `storage` variable is the number of notes the user would like to store in his account. And `source` is the Stripe token for the card that we are going to charge.

<li>We are using a `calculateCost(storage)` function (that we are going to add soon) to figure out how much to charge a user based on the number of notes that are going to be stored.

<li><li>We create a new Stripe object using our Stripe Secret key. We are getting this from the environment variable that we configured in the previous chapter.

<li>Finally, we use the `stripe.charges.create` method to charge the user and respond to the request if everything went through successfully.

Note, if you are testing this from India, you’ll need to add some shipping information as well. Check out the details from our forums.

## Add the Business Logic
Now let’s implement our `calculateCost` method. This is primarily our business logic.

 Create a `services/util/cost.js` and add the following.
```sh
export function calculateCost(storage) {
  const rate = storage <= 10 ? 4 : storage <= 100 ? 2 : 1;
  return rate * storage * 100;
}
```
This is basically saying that if a user wants to store 10 or fewer notes, we’ll charge them $4 per note. For 11 to 100 notes, we’ll charge $2 and any more than 100 is $1 per note. Since Stripe expects us to provide the amount in pennies (the currency’s smallest unit) we multiply the result by 100.

Clearly, our serverless infrastructure might be cheap but our service isn’t!

## Add the Route
Let’s add a new route for our billing API.

 Add the following below the `DELETE /notes/{id}` route in stacks/ApiStack.js.
```sh
"POST /billing": "functions/billing.main",
```
## Test the Billing API
Now that we have our billing API all set up, let’s do a quick test in our local environment.

We’ll be using the same CLI from a few chapters ago.

 Run the following in your terminal.
```sh
$ npx aws-api-gateway-cli-test \
--username='admin@example.com' \
--password='Passw0rd!' \
--user-pool-id='USER_POOL_ID' \
--app-client-id='USER_POOL_CLIENT_ID' \
--cognito-region='COGNITO_REGION' \
--identity-pool-id='IDENTITY_POOL_ID' \
--invoke-url='API_ENDPOINT' \
--api-gateway-region='API_REGION' \
--path-template='/billing' \
--method='POST' \
--body='{"source":"tok_visa","storage":21}'
```
Make sure to replace the `USER_POOL_ID`, `USER_POOL_CLIENT_ID`, `COGNITO_REGION`, I`DENTITY_POOL_ID`, `API_ENDPOINT`, and `API_REGION` with the same values we used a couple of chapters ago.

Here we are testing with a Stripe test token called `tok_visa` and with `21` as the number of notes we want to store. You can read more about the Stripe test cards and tokens in the Stripe API Docs here.

If the command is successful, the response will look similar to this.
```sh
Authenticating with User Pool
Getting temporary credentials
Making API request
{ status: 200, statusText: 'OK', data: { status: true } }
```
