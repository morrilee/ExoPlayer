# ExoPlayer <img src="https://img.shields.io/github/v/release/google/ExoPlayer.svg?label=latest"/>
This repo is a clone of the ExoPlayer repo.  It demonstrates the changes needed to play Amazon Music
Widevine DRM protected music using the default system Widevine L3 library.

## What changes were made

1) Added two Amazon Music samples to media.exolist.json, one encrypted (see below) and one unencrypted
2) Added serialize and unserialize of the license request and response from Amazon Music
3) set the USE_CRONET_FOR_NETWORKING=false to use the default network library to enable
   App Inspection --> Network Inspector
4) parse the license request url from `amz:LicenseUrl` in the manifest file
5) client passes in `User-Agent` to key-headers, remaining key-headers are hard coded
in the config file for testing

## media.exolist.json

The follow DRM protected sample asset is added
```
 {
    "name": "Amazon Music",
    "samples": [
      {
        "name": "Widevine DASH, OPUS/FLAC",
        "uri": "https://d17vo8z6jop21h.cloudfront.net/api/DashDrm.mpd?dmid=200000450871169&ld=false&sd=true&hd=false&uhd=false&3d=false&ra360=false&ac4=false&mhm1=false&bl=LOW:MEDIUM&v=opus_flac_e&drm=wv&r=b3879d0f-b0d0-493b-a131-be3ca01b8bc7&rgn=NA&dtype=A3RMGO6LYLH7YN",
        "drm_scheme": "widevine",
        "drm_license_uri": "[leave empty]",
        "drm_key_request_properties": {
          "Authorization": "bearer Atza|IwEBIDhrGR4GQYapkEmMAoAOX_AGwKQzU7MeJ3zJwImZ5J5XABcLqAWCj2HgNpUAwK5qWMEq5uaEsxaPSCl6r1t9pigCGVIE6BLqCx6CbDZwsxt3uWQiqs_Ws52QmXcFdE3WxGj-a9-vWMWmwPdhWbxsLVMJpmpU3nza8c9IM0bjZdUcOpbK8hoeT-WYZ-VRQSMUeMY",
          "x-amz-music-rid": "b3879d0f-b0d0-493b-a131-be3ca01b8bc7",
          "x-amz-music-asin": "B0B4TYM52Y",
          "x-amz-target": "com.amazon.digitalmusiclocator.DigitalMusicLocatorServiceExternal.getLicenseForPlaybackV3",
          "x-amz-music-device-type": "A3RMGO6LYLH7YN",
          "x-amz-music-device-id": "[TODO - device id]",
          "x-amz-music-customer-id": "[TODO - customer id]]",
          "x-amz-music-drm-type": "WIDEVINE",
          "Cookie": "rid=b3879d0f-b0d0-493b-a131-be3ca01b8bc7",
          "Content-Type": "application/json"
        }
      },
```

notes:

* ```drm_license_url``` This is left blank because the actual value is determined 
by the ```amz:LicenseURL``` in the manifest file

* ```drm_key_request_properties``` come from the key-headers in the PlayDirective if your media player
integrates with Amazon Music via Alexa Voice Services, or they come from key-headers
in the audio.object is using the Amazon Music SDK (https://developer.amazon.com/docs/music/API_browse_playback.html).

## Authorization

The license request needs to contain a auth token to authenticate the call.
The auth token is passed in the `Authorization` header.

* `Authorization` header contains the auth token to authenticate
  license request.  For voice clients this is included int the KeyHeaders.  For non-voice clients, this
  is the LWA token when the customer logs into Amazon Music.  NOTE: You need to replace the one in the
  code with a valid auth token.  The one in the code is expired which will result in a 401 for the license request.

## User-Agent

There is a client generated header `User-Agent` that needs to be passed in
as part of the key-headers.

`User-Agent` Required format: `<deviceName>/<deviceVersion> <appName>/<appVersion>``

ex: `Android/x.y ExoPlayerLib/2.18.7`

## Serialize license request

ExoPlayer obtains the license challenge as a binary blob.  The binary blob is base 64 encoded and
then encapulated by a json object.

```
{
   "licenseChallenge": [base64EncodedLicenseChallenge]
}
```

## Deserialize license response

ExoPlayer expects the license to be a binary blob.  The base64encoded license needs to be decoded
from the license response below.

```
{
   "license": "[base64EncodedLicense]"
}
```

## Other considerations

* if license request fails, then retry on 503 only with max retry of 1.  don't retry any other
failures / errors (e.g. 4xx, 500)
* for license request failures the response body contains more detailed information
including request id for further troubleshooting from the backend.  consider logging the request id
for better troubleshooting.
* the first 30s of all drm protected tracks is clear (i.e. not encrypted).  to reduce the initial 
playback delay consider playing the first three 10s segments while fetching the license in parallel.  
* the default license duration is 24h. to shorten this duration for testing you can pass
`x-amz-music-license-expiration-secs` http header into the license request
