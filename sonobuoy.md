# Running Sonobuoy conformance test when Docker rate limits

Output of `sonobuoy version`:

    Sonobuoy Version: v0.20.0
    MinimumKubeVersion: 1.17.0
    MaximumKubeVersion: 1.99.99
    GitSHA: f6e19140201d6bf2f1274bf6567087bc25154210
    API Version check skipped due to missing `--kubeconfig` or other error

Specify registry:

    REGISTRY=10.60.253.37/magnum

Generate `sonobuoy.yml` and modify as follows:

    cat << EOF > sonobuoy.yml
buildImageRegistry: k8s.gcr.io/build-image
dockerGluster: $REGISTRY #docker.io/gluster
dockerLibraryRegistry: $REGISTRY #docker.io/library
e2eRegistry: gcr.io/kubernetes-e2e-test-images
e2eVolumeRegistry: gcr.io/kubernetes-e2e-test-images/volume
gcRegistry: k8s.gcr.io
promoterE2eRegistry: k8s.gcr.io/e2e-test-images
sigStorageRegistry: k8s.gcr.io/sig-storage
EOF

Push the images from Docker Hub to a custom repo:

    sonobuoy images push --e2e-repo-config sonobuoy.yml --custom-registry $REGISTRY

Run conformance:

    sonobuoy run --e2e-repo-config sonobuoy.yml --sonobuoy-image $REGISTRY/sonobuoy:v0.20.0 --systemd-logs-image $REGISTRY/systemd-logs:v0.3