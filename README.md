
Workflows to generate attestations for checksum files, downloaded and verified from s3. The idea is that the user has some artifact from us, and the corresponding checksums.txt file.

Workflows can be triggered with a webhook, specifying the `VERSION`:

```bash
curl -L -X POST \
  -H "Accept: application/vnd.github+json" \
  -H "Authorization: Bearer <TOKEN>" \
  -H "X-GitHub-Api-Version: 2022-11-28" \
  https://api.github.com/repos/mveytsman/attestations-test/dispatches \
  -d '{"event_type":"attestation","client_payload": {"version": "1.2.3"}}'
```
## Option 1: Using Cosign

See [cosign.yml](https://github.com/mveytsman/attestations-test/blob/main/.github/workflows/cosign.yml)

1) Download `s3://mveytsman-test/${VERSION}/checksums.txt` and `s3://mveytsman-test/checksums.txt.sig`
2) Verify that the signature is valid for `checksums.txt`
3) Use `cosign` to generate an attestation for `checksums.txt` as `checksums.bundle`
4) Upload `checksums.bundle` to S3 at `s3://mveytsman-test/${VERSION}/checksums.txt.bundle

### Usage
The user will verify the checksums.txt with
```bash
cosign verify-blob checksums.txt  \
 --bundle checksums.txt.bundle \
 --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
 --certificate-identity "https://github.com/mveytsman/cosign-test/.github/workflows/cosign.yml@refs/heads/main"
```

They will then verify the artifacts match the checksums
```
sha256sum --ignore-missing -c checksums.txt
```

Only is the above two steps succeed will they proceed with the install.

## Option 2: Using GitHub's tooling

Github has [tooling](https://github.blog/changelog/2024-06-25-artifact-attestations-is-generally-available/) which wraps cosign

See [gh.yml](https://github.com/mveytsman/attestations-test/blob/main/.github/workflows/gh.yml)

1) Download `s3://mveytsman-test/${VERSION}/checksums.txt` and `s3://mveytsman-test/checksums.txt.sig`
2) Verify that the signature is valid for `checksums.txt`
3) Use github's attestation action to generate an attestation for `checksums.txt` as `checksums.bundle`

### Usage
The user will verify the checksums.txt with
```
gh attestation verify checksums.txt -R mveytsman/attestations-test
```

They will then verify the artifacts match the checksums
```
sha256sum --ignore-missing -c checksums.txt
```

Only is the above two steps succeed will they proceed with the install.


## Pros & Cons
GitHub's tooling saves us having to upload the bundle (which means the GitHub action token can have read-only s3 permissions), and is a little easier on the eyes. The downside is that it locks us more to github and less to generic cosign.
