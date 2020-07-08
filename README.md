# Code examples

## Javascript

```javascript
const getGithubAssetStatisticsRequests = (assets) => {
  const requests = assets.map(({ id, externalId }) =>
    requestF
      .get({
        uri: `https://www.cryptocompare.com/api/data/socialstats/?id=${externalId}`,
      })
      .chain((data) => Future.try(() => JSON.parse(data)))
      .map(path(["Data", "CodeRepository", "List"]))
      .map(pluck("url"))
      .map(when(isNil, always([])))
      .chain(getGithubStatistics.bind(null, id))
      .map(flatten)
      .map(map(filterNilFields))
      .map(filterIncompleteData)
      .chainRej((error) => {
        log.error("error while getting asset github statistics", {
          externalId,
          error,
        });

        return Future.of(null);
      })
      .map((data) => {
        log.info("statistics from github", { assetId: id, data });

        return data;
      })
  );

  return [assets, Future.parallel(coinsParallelRequestsLimit, requests)];
};
```

---

## Typescript

```typescript
async mergeImagesAndUpload(orderImage: OrderImage, frames: IFrame[]) {
    const { buffer: orderImageBuffer } = await getStream(orderImage.imageUri);
    const croppedOrderImageBuffer = await sharp(orderImageBuffer)
      .resize({
        width: 500,
      })
      .toBuffer();

    return Promise.all(
      frames.map(async frame => {
        const { buffer: frameBuffer } = await getStream(frame.backgroundImage);
        const croppedFrameBuffer = await sharp(frameBuffer)
          .resize({
            width: 500,
          })
          .toBuffer();
        const frameMeta = await sharp(croppedFrameBuffer).metadata();
        const resizedOrderImageBuffer = await sharp(croppedOrderImageBuffer)
          .resize({
            height: calcPercentsToPixels(frameMeta.height, frame.height),
            width: calcPercentsToPixels(frameMeta.width, frame.width),
          })
          .toBuffer();
        const newImageBuffer = await sharp(croppedFrameBuffer)
          .composite([
            {
              input: resizedOrderImageBuffer,
              left: calcPercentsToPixels(frameMeta.width, frame.left),
              top: calcPercentsToPixels(frameMeta.height, frame.top),
            },
          ])
          .toBuffer();
        const { url } = await this.fileManagementService.upload(
          { buffer: newImageBuffer, mimetype: 'image/png', originalname: 'tmp' },
          {
            folder: TMP_EMAIL_IMAGES_FOLDER,
          },
        );

        return {
          ...frame,
          backgroundImage: url,
          linkToFrame: `${frame.linkToFrame}&img=${orderImage.id}`,
        };
      }),
    );
  }
```
