# Image Verification

## Context

* The Update-Agent may only run Containers (and InitContainers) were it can make sure, that the Image is provided by KUKA.
* Image signatures shall not be used, as there is currently no infrastructure in-place, that distributes the signatures.
* As Bundle-Descriptions are already signed, we can ensure safe images, when we guarantee sameness of the used image to 
  the image used while creating the bundle.
* We cannot use the Manifest-Digest (which would be the same thing that the 
  [GPG signing process](https://github.com/containers/image/blob/main/docs/containers-signature.5.md#json-data-format) 
  because they are not stable when save/loading images from the USB-Stick.

## Decision

* We rely on the ImageID which is a hash of the individual layer-hashes of the image 
  (see [here](https://windsock.io/explaining-docker-image-ids/))
* This ImageID is stable even when not pulled from the originating Repository, but differs if layers are changed.

## Consequences

* Each Image-Reference in a Bundle-Description **must** have a corresponding ImageID present.
* After pulling an Image by Image-Reference we compare the local ImageID with the ImageID of the Bundle to ensure
  sameness.
* If the ImageIDs do not match the Bundle is invalid and may not be processed and containers based on its images may not
  be created.
* If an image is referenced by a tag and the image behind the tag changes all Bundles created with the previous ImageID 
  are automatically rendered invalid.