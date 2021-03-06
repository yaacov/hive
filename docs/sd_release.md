# Deploy OpenShift Hive for Service Delivery

## Description

The purpose of this doc is to show how to do a deployment of OpenShift Hive for Service Delivery.


## Build installer Image

- Setup some useful variables:
  ```shell
  > export RELEASE_VER=$(date +%Y%m%d)
  > export INSTALLER_GOPATH=/path/to/openshift/installer/gopath
  > export INSTALLER_SRC=${INSTALLER_GOPATH}/src/github.com/openshift/installer
  > export INSTALLER_IMAGE_TAG=quay.io/twiest/installer:${RELEASE_VER}
  ```

- Get the latest installer code
  ```shell
  > cd "${INSTALLER_SRC}"
  > git checkout master
  > git fetch upstream
  > git reset --hard upstream/master
  ```

- Build the installer container
  ```shell
  > cd "${INSTALLER_SRC}"
  > BUILDAH_ISOLATION=chroot sudo buildah bud --file images/installer/Dockerfile.ci --tag "${INSTALLER_IMAGE_TAG}" .
  ```

- Push the build installer image to quay
  ```shell
  > BUILDAH_ISOLATION=chroot sudo buildah push --authfile $HOME/.docker/config.json ${INSTALLER_IMAGE_TAG} docker://${INSTALLER_IMAGE_TAG}
  ```

## Build the hive-controller Image
- Setup some useful variables:
  ```shell
  > export HIVE_GOPATH=/path/to/openshift/installer/gopath
  > export HIVE_SRC=${HIVE_GOPATH}/src/github.com/openshift/hive
  > export HIVE_IMAGE_TAG=quay.io/twiest/hive-controller:${RELEASE_VER}
  ```

- Get the latest installer code
  ```shell
  > cd "${HIVE_SRC}"
  > git checkout master
  > git fetch upstream
  > git reset --hard upstream/master
  ```

- Build the installer container
  ```shell
  > cd "${HIVE_SRC}"
  > GOPATH="${HIVE_GOPATH}" make docker-build
  > docker tag hive-controller "${HIVE_IMAGE_TAG}"
  ```

- Push the build installer image to quay
  ```shell
  > docker push "${HIVE_IMAGE_TAG}"
  ```

## Change the OpenShift Hive SD Deploy to point to the new images
- Setup some useful variables:
  ```shell
  > export HIVE_SD_RELEASE_BRANCH="sd-release-${RELEASE_VER}"
  ```

- Create branch for PR:
  ```shell
  > cd "${HIVE_SRC}"
  > git checkout master
  > git fetch upstream
  > git reset --hard upstream/master
  > git checkout -b "${HIVE_SD_RELEASE_BRANCH}"
  > git push -u origin "${HIVE_SD_RELEASE_BRANCH}"
  ```

- Update image versions in Kustomize and README:
  ```shell
  > cd "${HIVE_SRC}"
  > sed -r -i -e "s%quay.io/twiest/hive-controller:[0-9]{8}%quay.io/twiest/hive-controller:${RELEASE_VER}%" README.md config/overlays/sd-dev/image_patch.yaml
  > sed -r -i -e "s%quay.io/twiest/installer:[0-9]{8}%quay.io/twiest/installer:${RELEASE_VER}%" README.md config/overlays/sd-dev/image_patch.yaml
  ```

- Commit and push changes
  ```shell
  > git commit -a -m "Update SD image links to latest release."
  > git push origin
  ```

- Create PR in github for the new release


## Deploy the new OpenShift Hive and Installer builds to the SD cluster
- Login using `oc` to the SD cluster (aka opshive)
- Remove all existing Hive objects (clusterdeployments, dnszones, etc). We're not currently trying to be backwards compatible, so this is necessary.
- Deploy the new OpenShift Hive build:
  ```shell
  > cd "${HIVE_SRC}"
  > GOPATH="${HIVE_GOPATH}" make deploy-sd-dev
  ```
- Test deploying a cluster using the instructions in the README. Make sure to use the instructions that use remote images and that the remote images are the new ones that were just built.

- Notify the SD guys (@Juan Hernandez @cben @Elad) in the `#service-development` channel that the new OpenShift Hive and Installer release is ready for them to consume.
