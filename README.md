
# SimpleSSAI (beta)

API-driven Server Side Ad Insertion

## About

SimpleSSAI provides Ad Stitching for VoD assets by calling an API and providing your source stream and Ad Server URL. Currently, we only handle the Ad Stitching piece, and not the Ad/Content transcoding(well, at least not yet )

Current ABR Streaming format support:
 - [x] HLS VoD
 - [ ] HLS VoD-To-Live w/ Channel Assembly (In Development)
 - [ ] HLS Live/Event Linear (Planned)
 - [ ] DASH VoD (Planned)
 - [ ] DASH Live (Planned)

## Content Conditions & Preparation
In order for Ad insertion to work for HLS, all Ads need to be encoded with the same encoding configuration used for main content - This is to allow for playlist alignment. This means using the same Encoding Ladder including resolutions/aspect-ratios, codecs, container types ,etc.

### SSAI HLS Packaging & Encoding:
The Ads and Main Content HLS Playlists should align - including:
 - **Video Variants** -> Ideally should have bitrates, codecs, codec profiles, resolutions
 - **Audio Media Playlists** -> (if using demuxed A/V) including same GROUP-ID, LANGUAGE, and NAME
 - **Subtitle Media Playlists** -> (if using) including same GROUP-ID, LANGUAGE,  and NAME
 - **I-FRAME Only Playlists** -> (if using)
 
Generally, using the same Encoder & Packager w/ the same configuration for both Main Content and Ads is recommended.

### SSAI Conditioning:
Ad segments are always stitched into the main content's Playlists at segment boundaries - it is not possible to stitch Ads within a segment. Therefore, when encoding your content assets, be sure to insert additional Keyframes with segment splits at all Ad opportunity time boundaries. For example, take a look at this Encoding script example from the Bitmovin Encoder -> [Bitmovin SSAI Conditions Example](https://github.com/bitmovin/bitmovin-api-sdk-examples/blob/main/javascript/src/ServerSideAdInsertion.ts). In this encoding script, additional Keyframes w/ segment splits are inserted at each time point of an Ad Opportunity. This is not an absolute requirement, but if not done, the SimpleSSAI stitcher will only be able insert Ad Segments at the closest segment boundary to provided AdBreak start time - and most likely not be exact. For the best accuracy in stitching Ads, HLS SpliceInfo Markers(`#EXT-X-CUE-OUT` and `#EXT-X-CUE-IN`) should be inserted at the Ad segment boundaries from the contribution encoder/packager.

### Ad Serving:
The Ad Decision Server should serve Ads in the form of VMAP responses that include Ad Breaks at desired `timeOffsets`. The Ad Breaks should be presented as VAST tags and each VAST document should contain a `MediaFile` that has a delivery type of `streaming` which links the URL of the HLS packaged Ad. This Ad, as described in the above *SSAI HLS Packaging & Encoding* section, should be encoded in the same manner as the content including the Encoding Bitrate Ladder, Audio Variants, Subtitle Variants, etc..  Ad Serving best practices:
 - VMAPs are most efficient when using InLine(`<InLine>`) VAST tags for the Ad Breaks.
	 - Wrapper/VASTAdTagURI are allowed but significantly increase response time.
	 - Max number of allowed `<Wrapper>` redirects is 7
 - Ads must have a corresponding MediaFile(`<MediaFile>`) with a **delivery** value of streaming
	 - For HLS packaged Ads, the MediaFile's **type** should be "application/x-mpegURL"
	 - Ads without the above delivery and type attributes will be discarded from the AdBreak sequence
 - AdPods and Buffet Ads are allowed. Buffet Ads are encouraged and used as fallback Ads.

### Examples:
VMAP with InLine AdBreaks carrying HLS Streaming Ads -> https://slightly-private.s3.amazonaws.com/test_assets/vmap_inline.xml
HLS Packaged Asset -> https://slightly-private.s3.amazonaws.com/SimpleSSAI_Content/2022-05-16T02%3A56%3A25.837Z/manifest.m3u8

API:
Endpoint: POST https://2ddkncp0bh.execute-api.us-east-2.amazonaws.com/Stage/hls
JSON Payload:
 - **hls_stream**: Stream URL for main content
 - **account_id**: UUID of the requesting Account. 
 - **strategy**: Segment insertion strategy Can be either `scteCue` or `closest`. Default is `closest`.
	 - `scteCue`: Most accurate. Combines using time from VMAP startOffset along with usage of `#EXT-X-CUE-OUT` and `#EXT-X-CUE-IN` HLS tags being present at Ad Boundaries in all Video variants and Audio/Subtitles MEDIAs
	 - `closest`: Uses closest segment boundary to given VMAP startOffset
 - **vmap_url**: Ad Decision Server URL that will return VMAP for the session.
 - **replaceContentDuration**: boolean `true` or `false` for whether or not main content segments should be replaced with Ad segments - instead of just inserted. Default is `false`

For testing your Streams/Ads/VMAPs, please use the below sample account_id. While testing, if you come across any issues, please fill out a GitHub Issue. Streams can encoded and packaged in many different ways, and it is very possible we have not yet taken into account all possibilities.

#### Request: POST: https://2ddkncp0bh.execute-api.us-east-2.amazonaws.com/Stage/hls
```
{
	"hls_stream": "https://slightly-private.s3.amazonaws.com/SimpleSSAI_Content/2022-05-16T02%3A56%3A25.837Z/manifest.m3u8",
	"account_id": "a84e3714-d96d-461c-90f5-0a888dc47cd4",
	"strategy": "scteCue",
	"vmap_url": "https://slightly-private.s3.amazonaws.com/test_assets/vmap_inline.xml",
	"replaceContentDuration":false
}
```
#### Response:
```
{
	"status": <successful|failed>,
	"manifest": <CloudFront hosted Manifest URL>,
	"vmapFetchTime": 0.250,
	"sessionExpiry": null,
	"adBreaks": [
        {
            "adBreakId": "preroll",
            "startOffset": 0,
            "duration": 61.04,
            "endOffset": 61.04,
            "stitchedAds": [
                {
                    "adId": "30001",
                    "adDuration": 15.16,
                    "adStartOffset": 0,
                    "endOffset": 15.16
                },
                {
                    "adId": "20002",
                    "adDuration": 15.120000000000001,
                    "adStartOffset": 15.16,
                    "endOffset": 30.28
                },
                {
                    "adId": "20004",
                    "adDuration": 30.76,
                    "adStartOffset": 30.28,
                    "endOffset": 61.040000000000006
                }
            ]
        },
        {
            "adBreakId": "midroll-1",
            "startOffset": 241.92,
            "duration": 61.04,
            "endOffset": 302.96,
            "stitchedAds": [
                {
                    "adId": "30001",
                    "adDuration": 15.16,
                    "adStartOffset": 241.92,
                    "endOffset": 257.08
                },
                {
                    "adId": "20002",
                    "adDuration": 15.120000000000001,
                    "adStartOffset": 257.08,
                    "endOffset": 272.2
                },
                {
                    "adId": "20004",
                    "adDuration": 30.76,
                    "adStartOffset": 272.2,
                    "endOffset": 302.96
                }
            ]
        },
        {
            "adBreakId": "postroll",
            "startOffset": 1010.0799999999999,
            "duration": 61.04,
            "endOffset": 1071.12,
            "stitchedAds": [
                {
                    "adId": "30001",
                    "adDuration": 15.16,
                    "adStartOffset": 1010.0799999999999,
                    "endOffset": 1025.24
                },
                {
                    "adId": "20002",
                    "adDuration": 15.120000000000001,
                    "adStartOffset": 1025.24,
                    "endOffset": 1040.36
                },
                {
                    "adId": "20004",
                    "adDuration": 30.76,
                    "adStartOffset": 1040.36,
                    "endOffset": 1071.12
                }
            ]
        }
    ],
	"errors": [],
	"warnings": []
}
```
