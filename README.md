Why are my node streams hanging / not ending / losing data?
===

They might be hanging because:

### You're returning a stream to a user, but you added a 'data' handler to it without also pausing it

Attaching a 'data' handler puts the stream into flowing mode.  If the caller doesn't immediately (in the current tick) attach data/end handlers, they'll start losing data and possibly the 'end' event.

Possibly very bad:

```js
	let bytesRead = 0;
	outputStream.on('data', function(data) {
		bytesRead += data.length;
	});
	return outputStream;
```

Probably right:

```js
	let bytesRead = 0;
	outputStream.on('data', function(data) {
		bytesRead += data.length;
	});
	// We attached a 'data' handler, but don't let that put us into
	// flowing mode yet, because the user hasn't attached their own
	// 'data' handler yet.
	outputStream.pause();
	return outputStream;
```


### You're listening to the wrong 'end' / 'finish' event

'end' fires when a `Readable` stops reading; 'finish' fires when a `Writable` stops writing.

https://nodejs.org/api/stream.html#stream_event_end

https://nodejs.org/api/stream.html#stream_event_finish - talks about the "`end()` method", which has nothing to do with the 'end' event!


### One or more cases forget to call `callback()` in your stream `Transform`

Your `_transform` and `_flush` implementations need to call `callback()`, optionally with an error or data, in all cases.  Not necessarily within the same tick, but it must be called.

Buggy code that results in a stream that sometimes doesn't fire 'end':

```js
	_flush(callback) {
		// Last block might not be full-size, and now that we know we've reached
		// the end, we handle it here.
		if(!this._buf.length) {
			return;
		}
		const crc = calculate(this._buf);
		if(crc !== this._crc) {
			callback(new Error("bad crc"));
		}
		this.push(this._buf);
		callback();
	}
```

Less-buggy code with the missing line added:

```js
	_flush(callback) {
		// Last block might not be full-size, and now that we know we've reached
		// the end, we handle it here.
		if(!this._buf.length) {
			callback(); // <------------------------
			return;
		}
		const crc = calculate(this._buf);
		if(crc !== this._crc) {
			callback(new Error("bad crc"));
		}
		this.push(this._buf);
		callback();
	}
```


### You're using a buggy library

If you tell [google-api-nodejs-client](https://github.com/google/google-api-nodejs-client) to upload a stream, the library might pipe your stream into the oauth2 token refresh request instead of the actual API request: https://github.com/google/google-api-nodejs-client/issues/260

You can "patch" it by subclassing the `OAuth2` implementation:

```js
const google = require('googleapis');
const OAuth2 = google.auth.OAuth2;

class FixedOAuth2 extends OAuth2 {
	// Work around https://github.com/google/google-api-nodejs-client/issues/260
	// by patching getRequestMetadata with something that never returns an
	// auth request to googleapis/lib/apirequest.js:createAPIRequest
	//
	// If we don't patch this, the buggy googleapis/google-auth-library interaction
	// will hang our program forever when we try to upload a file when our access token
	// is expired.  (createAPIRequest decides to pipe the stream into the auth request
	// instead of the subsequent request.)
	//
	// We could always refresh the access token ourselves, but we prefer to also
	// patch the buggy code to prevent bugs from compounding.
	getRequestMetadata(optUri, metadataCb) {
		const thisCreds = this.credentials;

		if(!thisCreds.access_token && !thisCreds.refresh_token) {
			return metadataCb(new Error('No access or refresh token is set.'), null);
		}

		// if no expiry time, assume it's not expired
		const expiryDate = thisCreds.expiry_date;
		const isTokenExpired = expiryDate ? expiryDate <= (new Date()).getTime() : false;

		if(thisCreds.access_token && !isTokenExpired) {
			thisCreds.token_type = thisCreds.token_type || 'Bearer';
			const headers = {'Authorization': thisCreds.token_type + ' ' + thisCreds.access_token};
			return metadataCb(null, headers, null);
		} else {
			return metadataCb(new Error('Access token is expired.'), null);
		}
	}
}
```

Note that you'll still need to make the calls to refresh the access token yourself.

You can use it with something like:

```js
	const oauth2Client = new FixedOAuth2(clientId, clientSecret, redirectUrl);
	const drive = google.drive({version: 'v2', auth: oauth2Client});
```
