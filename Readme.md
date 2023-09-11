# All Systems Go 2023

## Demo Prep

### Build KBS

```bash
pushd ./kbs
docker build -t kbs .
popd
```

### Deploy KBS

```bash
k create deploy kbs --image ghcr.io/mkulke/kbs:asg23-demo --port 8080
k expose deploy/kbs --type NodePort
```

### Encrypt Image

Build coco-keyprovider:

```bash
pushd ./coco-keyprovider
docker build -t coco-keyprovider .
popd
```

Create encryption key:

```bash
KEY_FILE="image_key"
head -c 32 /dev/urandom | openssl enc > "$KEY_FILE"
```

Run coco-keyprovider w/ mounted key file:

```bash
docker run -d -p 50000:50000 -v "${PWD}/${KEY_FILE}:/${KEY_FILE}" --name coco-keyprovider coco-keyprovider
```

Encrypt image with skopeo:

```bash
echo '{"key-providers": {"attestation-agent": {"grpc": "localhost:50000"}}}' > ocicrypt.conf
IMAGE_DESTINATION="ghcr.io/mkulke/busybox-encrypted:asg23-demo"
KEY_ID="default/key/busybox-encrypted"
KEYPROVIDER_PARAMS="provider:attestation-agent:keypath=/${KEY_FILE}::keyid=kbs:///${KEY_ID}::algorithm=A256GCM"
OCICRYPT_KEYPROVIDER_CONFIG="${PWD}/ocicrypt.conf" skopeo copy --insecure-policy --encryption-key "$KEYPROVIDER_PARAMS" docker://busybox:stable "docker://${IMAGE_DESTINATION}"
docker rm -f coco-keyprovider
```

Provision key to KBS:

```bash
cat "$KEY_FILE" | k exec -i deploy/kbs -- tee "/opt/confidential-containers/kbs/repository/${KEY_ID}" > /dev/null
```

## Demo

### Deploy Encrypted Image w/ kata-remote RuntimeClass

```bash
k apply -f busybox-encrypted.yaml
```

### Verify Encryption Configuration

Inspect image:

```bash
skopeo inspect docker://ghcr.io/mkulke/busybox-encrypted:asg23-demo | jless
```

Decode attestation-agent annotation:

```bash
echo "eyJraWQiOiJrYnM6Ly8vZGVmYXVsdC9rZXkvYnVzeWJveC1lbmNyeXB0ZWQiLCJ3cmFwcGVkX2RhdGEiOiIxaWkrSHdyTWthQWlUdnVCeVRDTzFTZGNKMDBaVks2VGJ5c1l3NStCM3lqQk8yS3hnckY0TjRDMWRmYlNUOGJDSE81ZVdwVGlVRTRXdkd1UWxZREpWYy9vdlVwSE5pdElnY1NNbHh6NVd3WndXQW85M3Q5TEs1V2hNb2RrcDdwakpqV0NScFBnZ0NWNytLeDdYMmVrb0dMRUVhNmdlNk5LSHlrZmhZUDlNRnRIMTVhYTdsM3RBSUtuVkhqYk9KSFhFY3NNRjBxYUVhK2VDWThLVGMyOEdENUpGZXpMbTlwelJ3ci93bWV2d08vdmVRRWlYSWhPR0NkUTRTWjlkeVEyOEVQUXBEYjFTUXBITnd0UXFrSUlWdWc9IiwiaXYiOiJ3Qi9PQ0ErSWpMVkhNL21MIiwid3JhcF90eXBlIjoiQTI1NkdDTSJ9" | base64 -d
```

### Check KBS logs

```bash
k logs deploy/kbs
```
