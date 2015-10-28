### Forms and non-Forms Functions

#### Retrieve session (install or open) parameters

These session parameters will be available at any point later on with this command. If no params, the dictionary will be empty. This refreshes with every new session (app installs AND app opens)
```csharp
Branch branch = Branch.GetInstance ();
Dictionary<string, object> sessionParams = branch.GetLatestReferringParams();
```

#### Retrieve install (install only) parameters

If you ever want to access the original session params (the parameters passed in for the first install event only), you can use this line. This is useful if you only want to reward users who newly installed the app from a referral link or something.
```csharp
Branch branch = Branch.GetInstance ();
Dictionary<string, object> installParams = branch.GetFirstReferringParams();
```

### Persistent identities

Often, you might have your own user IDs, or want referral and event data to persist across platforms or uninstall/reinstall. It's helpful if you know your users access your service from different devices. This where we introduce the concept of an 'identity'.

To identify a user, just call:
```csharp
Branch branch = Branch.GetInstance ();
branch.SetIdentityAsync("your user id", this);  // Where this implements IBranchIdentityInterface
```

#### Logout

If you provide a logout function in your app, be sure to clear the user when the logout completes. This will ensure that all the stored parameters get cleared and all events are properly attributed to the right identity.

**Warning** this call will clear the referral credits and attribution on the device.

```csharp
Branch.GetInstance(getApplicationContext()).LogoutAsync(this); // Where this implements IBranchIdentityInterface
```

### Register custom events

```csharp
Branch branch = Branch.GetInstance ();
await branch.UserCompletedActionAsync("your_custom_event");
```

OR if you want to store some state with the event

```csharp
Branch branch = Branch.GetInstance ();
Dictionary<string, object> data = new Dictionary<string, object>();
data.Add("sku", "123456789");
await branch.UserCompletedActionAsync("purchase_event", data);
```

Some example events you might want to track:
```csharp
"complete_purchase"
"wrote_message"
"finished_level_ten"
```

## Generate Tracked, Deep Linking URLs (pass data across install and open)

### Shortened links

There are a bunch of options for creating these links. You can tag them for analytics in the dashboard, or you can even pass data to the new installs or opens that come from the link click. How awesome is that? You need to pass a callback for when you link is prepared (which should return very quickly, ~ 50 ms to process).

For more details on how to create links, see the [Branch link creation guide](https://github.com/BranchMetrics/Branch-Integration-Guides/blob/master/url-creation-guide.md)

```csharp
// associate data with a link
// you can access this data from any instance that installs or opens the app from this link (amazing...)

var data = new Dictionary<string, object>(); 
data.Add("user", "Joe");
data.Add("profile_pic", "https://s3-us-west-1.amazonaws.com/myapp/joes_pic.jpg");
data.Add("description", "Joe likes long walks on the beach...") 

// associate a url with a set of tags, channel, feature, and stage for better analytics.
// tags: null or example set of tags could be "version1", "trial6", etc
// channel: null or examples: "facebook", "twitter", "text_message", etc
// feature: null or examples: Branch.FEATURE_TAG_SHARE, Branch.FEATURE_TAG_REFERRAL, "unlock", etc
// stage: null or examples: "past_customer", "logged_in", "level_6"

List<String> tags = new List<String>();
tags.Add("version1");
tags.Add("trial6");

// Link 'type' can be used for scenarios where you want the link to only deep link the first time. 
// Use _null_, _LINK_TYPE_UNLIMITED_USE_ or _LINK_TYPE_ONE_TIME_USE_

// Link 'alias' can be used to label the endpoint on the link. For example: http://bnc.lt/AUSTIN28. 
// Be careful about aliases: these are immutable objects permanently associated with the data and associated paramters you pass into the link. When you create one in the SDK, it's tied to that user identity as well (automatically specified by the Branch internals). If you want to retrieve the same link again, you'll need to call getShortUrl with all of the same parameters from before.

Branch branch = Branch.GetInstance ();
await branch.GetShortUrlAsync(this, data, "alias","channel","stage", tags, "feature", uriType);

// The error method of the callback will be called if the link generation fails (or if the alias specified is aleady taken.)
```

There are other methods which exclude tags and data if you don't want to pass those. Explore the autocomplete functionality.

**Note**
You can customize the Facebook OG tags of each URL if you want to dynamically share content by using the following _optional keys in the data dictionary_. Please use this [Facebook tool](https://developers.facebook.com/tools/debug/og/object) to debug your OG tags!

| Key | Value
| --- | ---
| "$og_title" | The title you'd like to appear for the link in social media
| "$og_description" | The description you'd like to appear for the link in social media
| "$og_image_url" | The URL for the image you'd like to appear for the link in social media
| "$og_video" | The URL for the video 
| "$og_url" | The URL you'd like to appear
| "$og_app_id" | Your OG app ID. Optional and rarely used.

Also, you do custom redirection by inserting the following _optional keys in the dictionary_:

| Key | Value
| --- | ---
| "$desktop_url" | Where to send the user on a desktop or laptop. By default it is the Branch-hosted text-me service
| "$android_url" | The replacement URL for the Play Store to send the user if they don't have the app. _Only necessary if you want a mobile web splash_
| "$ios_url" | The replacement URL for the App Store to send the user if they don't have the app. _Only necessary if you want a mobile web splash_
| "$ipad_url" | Same as above but for iPad Store
| "$fire_url" | Same as above but for Amazon Fire Store
| "$blackberry_url" | Same as above but for Blackberry Store
| "$windows_phone_url" | Same as above but for Windows Store

You have the ability to control the direct deep linking of each link by inserting the following _optional keys in the dictionary_:

| Key | Value
| --- | ---
| "$deeplink_path" | The value of the deep link path that you'd like us to append to your URI. For example, you could specify "$deeplink_path": "radio/station/456" and we'll open the app with the URI "yourapp://radio/station/456?link_click_id=branch-identifier". This is primarily for supporting legacy deep linking infrastructure. 
| "$always_deeplink" | true or false. (default is not to deep link first) This key can be specified to have our linking service force try to open the app, even if we're not sure the user has the app installed. If the app is not installed, we fall back to the respective app store or $platform_url key. By default, we only open the app if we've seen a user initiate a session in your app from a Branch link (has been cookied and deep linked by Branch)

## Referral system rewarding functionality

In a standard referral system, you have 2 parties: the original user and the invitee. Our system is flexible enough to handle rewards for all users. Here are a couple example scenarios:

1) Reward the original user for taking action (eg. inviting, purchasing, etc)

2) Reward the invitee for installing the app from the original user's referral link

3) Reward the original user when the invitee takes action (eg. give the original user credit when their the invitee buys something)

These reward definitions are created on the dashboard, under the 'Reward Rules' section in the 'Referrals' tab on the dashboard.

Warning: For a referral program, you should not use unique awards for custom events and redeem pre-identify call. This can allow users to cheat the system.

### Get reward balance

Reward balances change randomly on the backend when certain actions are taken (defined by your rules), so you'll need to make an asynchronous call to retrieve the balance. Here is the syntax:

```csharp
Branch branch = Branch.GetInstance ();
await branch.LoadRewardsAsync(this);

#region IBranchRewardsInterface implementation

		public void RewardsLoaded ()
		{
			Device.BeginInvokeOnMainThread (() => {
				// Do something with the data...
			});
		}

		public void RewardsRedeemed (string bucket, int count)
		{
			Device.BeginInvokeOnMainThread (() => {
				// Do something with the data...
			});
		}

		public void CreditHistory (List<CreditHistoryEntry> history)
		{
			Device.BeginInvokeOnMainThread (() => {
				// Do something with the data...
			});
		}

		public void RewardsRequestError (BranchError error)
		{
			Device.BeginInvokeOnMainThread (() => {
				// Do something with the data...
			});
		}

#endregion
```

### Redeem all or some of the reward balance (store state)

We will store how many of the rewards have been deployed so that you don't have to track it on your end. In order to save that you gave the credits to the user, you can call redeem. Redemptions will reduce the balance of outstanding credits permanently.

```csharp
Branch branch = Branch.GetInstance ();
await branch.RedeemRewardsAsync(this, amount, bucket);

#region IBranchRewardsInterface implementation

		public void RewardsLoaded ()
		{
			Device.BeginInvokeOnMainThread (() => {
				// Do something with the data...
			});
		}

		public void RewardsRedeemed (string bucket, int count)
		{
			Device.BeginInvokeOnMainThread (() => {
				// Do something with the data...
			});
		}

		public void CreditHistory (List<CreditHistoryEntry> history)
		{
			Device.BeginInvokeOnMainThread (() => {
				// Do something with the data...
			});
		}

		public void RewardsRequestError (BranchError error)
		{
			Device.BeginInvokeOnMainThread (() => {
			 // Do something with the data...
			});
		}

		#endregion

```

### Get credit history

This call will retrieve the entire history of credits and redemptions from the individual user.  It also implements the IBranchRewardsInterface(see above). To use this call, implement like so:

```csharp
Branch branch = Branch.GetInstance ();
await branch.GetCreditHistoryAsync(this);

```

The response will return an array that has been parsed from the following JSON:
```json
[
    {
        "transaction": {
                           "date": "2014-10-14T01:54:40.425Z",
                           "id": "50388077461373184",
                           "bucket": "default",
                           "type": 0,
                           "amount": 5
                       },
        "referrer": "12345678",
        "referree": null
    },
    {
        "transaction": {
                           "date": "2014-10-14T01:55:09.474Z",
                           "id": "50388199301710081",
                           "bucket": "default",
                           "type": 2,
                           "amount": -3
                       },
        "referrer": null,
        "referree": "12345678"
    }
]
```
**referrer**
: The id of the referring user for this credit transaction. Returns null if no referrer is involved. Note this id is the user id in developer's own system that's previously passed to Branch's identify user API call.

**referree**
: The id of the user who was referred for this credit transaction. Returns null if no referree is involved. Note this id is the user id in developer's own system that's previously passed to Branch's identify user API call.

**type**
: This is the type of credit transaction

1. _0_ - A reward that was added automatically by the user completing an action or referral
1. _1_ - A reward that was added manually
2. _2_ - A redemption of credits that occurred through our API or SDKs
3. _3_ - This is a very unique case where we will subtract credits automatically when we detect fraud

### Get referral code

Retrieve the referral code created by current user

```csharp
Branch branch = Branch.GetInstance ();
await branch.GetReferralCodeAsync(this, amount);

#region IBranchReferralInterface implementation

		public void ReferralCodeCreated (string code)
		{
			Device.BeginInvokeOnMainThread (() => {
				// Do something with the data...
			});
		}

		public void ReferralCodeValidated (string code, bool valid)
		{
			Device.BeginInvokeOnMainThread (() => {
				// Do something with the data...
			});
		}

		public void ReferralCodeApplied (string code)
		{
			Device.BeginInvokeOnMainThread (() => {
				// Do something with the data...
			});
		}

		public void ReferralRequestError (BranchError error)
		{
			Device.BeginInvokeOnMainThread (() => {
				// Do something with the data...
			});
		}

#endregion

		

```

### Create referral code

Create a new referral code for the current user, only if this user doesn't have any existing non-expired referral code.

In the simplest form, just specify an amount for the referral code.
The returned referral code is a 6 character long unique alpha-numeric string wrapped inside the params dictionary with key @"referral_code".

**amount** _int_
: The amount of credit to redeem when user applies the referral code

```csharp
// Create a referral code of 5 credits 

Branch branch = Branch.GetInstance();
await branch.GetReferralCodeAsync(this, 5);

#region IBranchReferralInterface implementation

		public void ReferralCodeCreated (string code)
		{
			Device.BeginInvokeOnMainThread (() => { 
				// Do something with the data...
			});
		}
...

#endregion
```

Alternatively, you can specify a prefix for the referral code.
The resulting code will have your prefix, concatenated with a 4 character long unique alpha-numeric string wrapped in the same data structure.

**prefix** _String_
: The prefix to the referral code that you desire

```csharp
Branch branch = Branch.GetInstance();
await branch.GetReferralCodeAsync(this, 5, "BRANCH");

#region IBranchReferralInterface implementation

		public void ReferralCodeCreated (string code)
		{
			Device.BeginInvokeOnMainThread (() => { 
				// Do something with the data...
			});
		}
...

#endregion 
```

If you want to specify an expiration date for the referral code, you can add an expiration parameter.
The prefix parameter is optional here, i.e. it could be getReferralCode(5, expirationDate, new BranchReferralInitListener()...

**expiration** _Date_
: The expiration date of the referral code

```csharp
Branch branch = Branch.GetInstance();
await branch.GetReferralCodeAsync(this, 5, "BRANCH", expirationDate);

#region IBranchReferralInterface implementation

		public void ReferralCodeCreated (string code)
		{
			Device.BeginInvokeOnMainThread (() => { 
				// Do something with the data...
			});
		}
...

#endregion  
```

You can also tune the referral code to the finest granularity, with the following additional parameters:

**bucket** _String_
: The name of the bucket to use. If none is specified, defaults to 'default'

**calculation_type**  _int_
: This defines whether the referral code can be applied indefinitely, or only once per user

1. _REFERRAL_CODE_AWARD_UNLIMITED_ - referral code can be applied continually
1. _REFERRAL_CODE_AWARD_UNIQUE_ - a user can only apply a specific referral code once

**location** _int_
: The user to reward for applying the referral code

1. _REFERRAL_CODE_LOCATION_REFERREE_ - the user applying the referral code receives credit
1. _REFERRAL_CODE_LOCATION_REFERRING_USER_ - the user who created the referral code receives credit
1. _REFERRAL_CODE_LOCATION_BOTH_ - both the creator and applicant receive credit

```csharp
Branch branch = Branch.GetInstance();
await branch.GetReferralCodeAsync(this, 5, "BRANCH", expirationDate, "default", Constants.REFERRAL_CODE_AWARD_UNLIMITED,Constants.REFERRAL_CODE_LOCATION_REFERRING_USER);

#region IBranchReferralInterface implementation

		public void ReferralCodeCreated (string code)
		{
			Device.BeginInvokeOnMainThread (() => { 
				// Do something with the data...
			});
		}
...

#endregion  
```

### Validate referral code

Validate if a referral code exists in Branch system and is still valid.
A code is vaild if:

1. It hasn't expired.
1. If its calculation type is uniqe, it hasn't been applied by current user.

If valid, returns the referral code JSONObject in the call back.

**code** _String_
: The referral code to validate

```csharp
Branch branch = Branch.GetInstance();
await branch.ValidateReferralCodeAsync(this, "code");

#region IBranchReferralInterface implementation

		...

		public void ReferralCodeValidated (string code, bool valid)
		{
			Device.BeginInvokeOnMainThread (() => {
				// Do something with the data...
			});
		}

		...

#endregion
 
```

### Apply referral code

Apply a referral code if it exists in Branch system and is still valid (see above).
If the code is valid, returns the referral code JSONObject in the call back.

**code** _String_
: The referral code to apply

```csharp

Branch branch = Branch.GetInstance();
await branch.ApplyReferralCodeAsync(this, "code");

#region IBranchReferralInterface implementation

		...

		public void ReferralCodeApplied (string code)
		{
			Device.BeginInvokeOnMainThread (() => {
			// Do something with the data...
			});
		}

		...

#endregion

```