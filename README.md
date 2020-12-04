# Apple airport parser
Python script to turn the output of the `airport` utility to more manageable JSON format with some fields filtered out.

## TL;DR
This script is result of the necessity to scan the wireless networks on macOS from terminal and the need to consume the result by some other components understanding JSON.

### Usage
```bash
$ /System/Library/PrivateFrameworks/Apple80211.framework/Resources/airport -xs | ./airport-parser # | jq dump
$ # OR
$ ./airport-parser ~/path.to.airport.dump.xml
$ # Output
[
  {
    "HT": true,
    "radio": {
      "rssi": -39,
      "noise": -95,
      "channel": {
        "current": 36,
        "flags": 1040
      },
      "mode": 2,
      "country": "CZ"
    },
    "bssid": "XX:XX:XX:XX:XXX",
    "ssid": "Next door 5G",
    "security": [
      {
        "type": "wpa2",
        "auth": [
          "psk"
        ],
        "unicast": [
          "ccmp"
        ],
        "group": "ccmp"
      }
    ]
  }
]
```

## Story time
When dealing with some terminal wizardery I needed to find out all wireless networks in the vacinity. The Apple have a tool for probing the wireless networks "hidden" in private framework: `/System/Library/PrivateFrameworks/Apple80211.framework/Resources/airport`. The tool outputs scan data (using `airport -s`) such as:

```bash
                            SSID BSSID             RSSI CHANNEL HT CC SECURITY (auth/unicast/group)
                XNet @ Protected XX:XX:XX:XX:XX:XX -57  13      Y  CZ WPA2(PSK/AES/AES) 
                   XNet @ Guests XX:XX:XX:XX:XX:XX -59  13      Y  CZ WPA2(PSK/AES/AES) 
                     XNet @ Hall XX:XX:XX:XX:XX:XX -59  13      Y  CZ WPA2(PSK/AES/AES) 
                       Next door XX:XX:XX:XX:XX:XX -33  11      Y  -- WPA2(PSK/AES/AES) 
                        Internet XX:XX:XX:XX:XX:XX -65  6       Y  -- NONE

```

The first approach is to try to naively parse it "by hand" - using good old bash and tools - somewhere along the lines of (this code: is quite old; is horrible :facepalm: - but can be used as proof of concept for anyone interested - so here you go):
```bash
EXEC_AIR="/System/Library/PrivateFrameworks/Apple80211.framework/Resources/airport"
WIFIS=$($EXEC_AIR -s | awk -F. '\
   function ltrim(s) {sub(/^[ \t\r\n]+/, "", s); return s};  \
   function rtrim(s) {sub(/[ \t\r\n]+$/, "", s); return s}; \
   function trim(s)  { return rtrim(ltrim(s)); };            \
   \
   BEGIN  {FIELDWIDTHS = "33 18 5 8 3 3 80"; print "[\n"} \
   NR > 1 {print "    { \"ssid\": \"" trim($1) "\",", "\"rssi\": \"" trim($3) "\", \"security\": \"" trim($7) "\" },"} \
   END    {print "    { \"ssid\": \"" trim($1) "\",", "\"rssi\": \"" trim($3) "\", \"security\": \"" trim($7) "\" }\n]"}' \
   | jq 'sort_by(.ssid,.rssi) | unique_by(.ssid)')

echo $(jq -nc --argjson list "$WIFIS" '{Available: ($list)')
```

Apple provided the `-x` option allowing the output to be spitted out in XML (PLIST) format but `plutil` didn't take it as an input (abbrevieted version for one interface):

<details>
<summary>XML output example</summary>

```xml
<array>
	<dict>
		<key>80211D_IE</key>
		<dict>
			<key>IE_KEY_80211D_CHAN_INFO_ARRAY</key>
			<array>
				<dict>
					<key>IE_KEY_80211D_FIRST_CHANNEL</key>
					<integer>36</integer>
					<key>IE_KEY_80211D_MAX_POWER</key>
					<integer>20</integer>
					<key>IE_KEY_80211D_NUM_CHANNELS</key>
					<integer>1</integer>
				</dict>
			</array>
			<key>IE_KEY_80211D_COUNTRY_CODE</key>
			<string>CZ</string>
		</dict>
		<key>AGE</key><integer>0</integer>
		<key>AP_MODE</key><integer>2</integer>
		<key>BEACON_INT</key><integer>100</integer>
		<key>BSSID</key><string>XX:XX:XX:XX:XX:XX</string>
		<key>CAPABILITIES</key><integer>2321</integer>
		<key>CHANNEL</key><integer>48</integer>
		<key>CHANNEL_FLAGS</key><integer>20</integer>
		<key>HT_CAPS_IE</key>
		<dict>
			<key>AMPDU_PARAMS</key>
			<integer>23</integer>
			<key>ASEL_CAPS</key>
			<integer>0</integer>
			<key>CAPS</key>
			<integer>111</integer>
			<key>EXT_CAPS</key>
			<integer>0</integer>
			<key>MCS_SET</key><data> ... </data>
			<key>TXBF_CAPS</key><integer>0</integer>
		</dict>
		<key>HT_IE</key>
		<dict>
			<key>HT_BASIC_MCS_SET</key><data> ... </data>
			<key>HT_DUAL_BEACON</key><false/>
			<key>HT_DUAL_CTS_PROT</key><false/>
			<key>HT_LSIG_TXOP_PROT_FULL</key><false/>
			<key>HT_NON_GF_STAS_PRESENT</key><true/>
			<key>HT_OBSS_NON_HT_STAS_PRESENT</key><false/>
			<key>HT_OP_MODE</key><integer>0</integer>
			<key>HT_PCO_ACTIVE</key><false/>
			<key>HT_PCO_PHASE</key><false/>
			<key>HT_PRIMARY_CHAN</key><integer>48</integer>
			<key>HT_PSMP_STAS_ONLY</key><false/>
			<key>HT_RIFS_MODE</key><false/>
			<key>HT_SECONDARY_BEACON</key><false/>
      <key>HT_SECONDARY_CHAN_OFFSET</key><integer>3</integer>
      <key>HT_SERVICE_INT</key><integer>0</integer>
      <key>HT_STA_CHAN_WIDTH</key><true/>
      <key>HT_TX_BURST_LIMIT</key><false/>
		</dict>
		<key>IE</key><data> ... </data>
		<key>NOISE</key><integer>0</integer>
		<key>RATES</key><array><integer>6</integer></array>
		<key>RSN_IE</key>
		<dict>
			<key>IE_KEY_RSN_AUTHSELS</key>
			<array><integer>2</integer></array>
			<key>IE_KEY_RSN_MCIPHER</key><integer>4</integer>
			<key>IE_KEY_RSN_UCIPHERS</key><array><integer>4</integer></array>
			<key>IE_KEY_RSN_VERSION</key><integer>1</integer>
		</dict>
		<key>RSSI</key><integer>-58</integer>
		<key>SSID</key><data>...</data>
		<key>SSID_STR</key><string>XNet 5G @ Protected</string>
		<key>VHT_CAPS</key>
		<dict>
			<key>INFO</key><integer>843055152</integer>
			<key>SUPPORTED_MCS_SET</key><data> ... </data>
		</dict>
		<key>VHT_OP</key>
		<dict>
			<key>BASIC_MCS_SET</key><integer>-6</integer>
			<key>CHANNEL_CENTER_FREQUENCY_SEG0</key><integer>0</integer>
			<key>CHANNEL_CENTER_FREQUENCY_SEG1</key><integer>0</integer>
			<key>CHANNEL_WIDTH</key><integer>0</integer>
		</dict>
	</dict>
  <array>
```
</details>


After searching for some while and finding the coresponding `RNS IE` and `WPA IE` values for encryptions, the scirpt in this repository was born. It outputs information similar to the default `airport -s` only in `JSON`:
```json
[
  {
    "HT": true,
    "radio": {
      "rssi": -39,
      "noise": -95,
      "channel": {
        "current": 36,
        "flags": 1040
      },
      "mode": 2,
      "country": "CZ"
    },
    "bssid": "XX:XX:XX:XX:XXX",
    "ssid": "Next door 5G",
    "security": [
      {
        "type": "wpa2",
        "auth": [
          "psk"
        ],
        "unicast": [
          "ccmp"
        ],
        "group": "ccmp"
      }
    ]
  }
]
```
