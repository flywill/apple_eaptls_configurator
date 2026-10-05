# Apple EAP-TLS Mobileconfig Configurator

A simple, standalone web tool for generating an Apple `.mobileconfig` Wi-Fi profile for **802.1X / EAP-TLS** authentication.

The tool combines:

- a trusted **CA certificate** used to validate the RADIUS server,
- a **PKCS#12 client identity** (`.p12` / `.pfx`) containing the client certificate and private key,
- the target **Wi-Fi SSID**, and
- optional **RADIUS server name validation**

into a downloadable Apple configuration profile that can be installed on **iPhone, iPad, and macOS**.

## What it generates

The generated `.mobileconfig` contains three payloads:

1. **Root CA payload**
   - Installs the CA certificate used to validate the RADIUS server certificate.

2. **PKCS#12 identity payload**
   - Installs the client certificate and private key.
   - Includes the PKCS#12 password so the Apple device can import the identity.

3. **Managed Wi-Fi payload**
   - Configures the SSID.
   - Uses **EAP-TLS only** (EAP type 13).
   - References the installed client identity.
   - Anchors RADIUS trust to the supplied CA.
   - Can optionally restrict trust to a specific RADIUS server name.

## Privacy

Everything runs **locally in your browser**.

The page uses the browser FileReader API to read the selected certificate files and builds the configuration profile in memory.

**No certificate, private key, password, SSID, or generated profile is uploaded to a server by this tool.**

You can also download the HTML file and open it directly from your computer.

## Usage

Open the configurator in a browser and provide the following information.

### 1. CA certificate

Select the CA certificate that signed the certificate presented by your RADIUS server.

Supported formats:

- PEM: `.pem`, `.crt`, `.cer`
- DER: `.der`

Example:

```
ca.pem
```

### 2. Client identity

Select a PKCS#12 bundle containing the user's:

- client certificate
- private key

Supported extensions:

```
.p12
.pfx
```

Enter the password protecting the PKCS#12 file.

### 3. Wi-Fi SSID

Enter the Wi-Fi network name exactly as configured on the WLAN.

802.11 SSIDs are limited to **32 bytes**.

### 4. RADIUS server name

This field is optional, but using it is recommended.

Example:

```
radius.example.com
```

When configured, the Apple device will require the RADIUS server certificate to:

- chain to the CA included in the profile, and
- match the configured trusted server name.

The name should normally match the DNS name in the RADIUS server certificate's **Subject Alternative Name (SAN)**.

You can inspect a server certificate with:

```bash
openssl x509 -in server.pem -noout -subject -ext subjectAltName
```

If this field is left blank, the device can trust any RADIUS server certificate issued by the supplied CA.

### 5. User / identity label

Enter a label used to identify the profile and client identity.

Example:

```
alice
```

This is primarily used for display names, payload identifiers, and the generated filename.

### 6. Optional settings

The page also supports:

- Auto-join
- Hidden SSID
- Organization name
- Allowing trust exceptions for unknown RADIUS certificates

For normal enterprise EAP-TLS deployments, **do not enable trust exceptions** unless you specifically need that behavior.

## Building a PKCS#12 file

If you have a client certificate and private key separately, combine them into a PKCS#12 file first:

```bash
openssl pkcs12 -export \
  -in client.crt \
  -inkey client.key \
  -certfile ca.pem \
  -name client \
  -out client.p12
```

## Apple PKCS#12 compatibility

Some PKCS#12 files created with newer OpenSSL defaults use modern encryption algorithms that may work on newer iOS releases but fail to import on macOS or older iOS versions.

The configurator checks for several common incompatible encodings and will warn when it detects one.

If necessary, re-encode the PKCS#12 file using legacy 3DES encryption and a SHA-1 MAC:

```bash
openssl pkcs12 -in old.p12 -nodes -out flat.pem

openssl pkcs12 -export \
  -in flat.pem \
  -out new.p12 \
  -certpbe PBE-SHA1-3DES \
  -keypbe PBE-SHA1-3DES \
  -macalg sha1

rm flat.pem
```

Use the newly generated `new.p12` in the configurator.

## Resulting EAP configuration

The generated Wi-Fi payload uses:

```
AcceptEAPTypes = 13
```

which means **EAP-TLS only**.

The Wi-Fi payload references:

- the installed PKCS#12 client identity,
- the supplied CA as the trusted certificate anchor, and
- the optional trusted RADIUS server name.

This prevents the device from silently falling back to password-based EAP methods.

## Installing the profile

After generating the profile, download the resulting:

```
<SSID>-<user>.mobileconfig
```

Then transfer it to the Apple device.

Typical options include:

- AirDrop
- email
- a secure internal web server
- Apple Configurator
- MDM

On iPhone or iPad, opening the file will normally place the downloaded profile in the device's profile installation workflow.

On macOS, open the `.mobileconfig` file and follow the profile installation prompts.

## Security warning

**Treat the generated `.mobileconfig` file like a private key.**

The profile contains:

- the client's private key,
- the client certificate, and
- the PKCS#12 password required to import the identity.

Anyone who obtains the completed profile may be able to use that client identity.

Recommended handling:

1. Generate the profile on a trusted computer.
2. Transfer it to the intended device using a secure method.
3. Install the profile.
4. Delete unnecessary copies of the generated `.mobileconfig`.

For larger production deployments, use a proper MDM / certificate enrollment system instead of manually distributing profiles containing private keys.

## Running locally

No web server is required.

Clone the repository:

```bash
git clone https://github.com/flywill/apple_eaptls_configurator.git
cd apple_eaptls_configurator
```

Then open:

```
index.html
```

in a browser.

Because the application is a standalone HTML page, it can also be hosted as a static site.

## Repository files

```
index.html
mobileconfig-builder.html
```

Both currently contain the standalone configurator.

## Intended use

This project is useful for:

- EAP-TLS lab testing
- WLAN troubleshooting
- validating Apple EAP-TLS behavior
- small test deployments
- generating profiles for iOS, iPadOS, and macOS clients
- reproducing enterprise Wi-Fi authentication scenarios

It is especially useful in lab environments where manually creating Apple configuration profiles is inconvenient.

## Disclaimer

This project is provided as a utility for testing and configuration purposes.

Review the generated profile and your organization's certificate/security requirements before using it in production.

## License

No license has been specified yet.
