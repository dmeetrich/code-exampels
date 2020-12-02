# Code examples

## Javascript

```javascript
const getContributorsScheme = (id, repo, data) =>
  applySpec([
    {
      assetId: always(id),
      name: always("contributors"),
      title: always("contributors"),
      resourceId: always(repo),
      website: always("github"),
      value: length,
      updateDateTime: always(moment().valueOf()),
    },
    {
      assetId: always(id),
      name: always("commits"),
      title: always("commits"),
      resourceId: always(repo),
      website: always("github"),
      value: pipe(pluck("total"), sum),
      updateDateTime: always(moment().valueOf()),
    },
  ])(data);

const getReleasesScheme = (id, repo, data) =>
  applySpec({
    assetId: always(id),
    name: always("releases"),
    title: always("releases"),
    resourceId: always(repo),
    website: always("github"),
    value: length,
    updateDateTime: always(moment().valueOf()),
  })(data);

const getCodeFrequencyScheme = (id, repo, data) =>
  applySpec([
    {
      assetId: always(id),
      name: always("week_adds"),
      title: always("week_adds"),
      resourceId: always(repo),
      website: always("github"),
      value: ifElse(isEmpty, always(null), pipe(last, nth(1))),
      updateDateTime: always(moment().valueOf()),
    },
    {
      assetId: always(id),
      name: always("week_dels"),
      title: always("week_dels"),
      resourceId: always(repo),
      website: always("github"),
      value: ifElse(isEmpty, always(null), pipe(last, nth(2))),
      updateDateTime: always(moment().valueOf()),
    },
  ])(data);

const getParticipationScheme = (id, repo, data) =>
  applySpec({
    assetId: always(id),
    name: always("week_commits"),
    title: always("week_commits"),
    resourceId: always(repo),
    website: always("github"),
    value: ifElse(isEmpty, always(null), pipe(prop("all"), sum)),
    updateDateTime: always(moment().valueOf()),
  })(data);

const getPunchCardScheme = (id, repo, data) => {
  const commitsPerHours = ifElse(
    isEmpty,
    always(null),
    pipe(reverse, take(24))
  )(data);

  if (!(commitsPerHours && commitsPerHours.length)) {
    return [];
  }

  return commitsPerHours.map((commitsPerHour) => {
    const hour = nth(1)(commitsPerHour);

    return applySpec({
      assetId: always(id),
      name: always(`hour_${hour}_commits`),
      title: always(`hour_${hour}_commits`),
      resourceId: always(repo),
      website: always("github"),
      value: last,
      updateDateTime: always(moment().valueOf()),
    })(commitsPerHour);
  });
};

const apis = {
  "/stats/contributors": getContributorsScheme,
  "/releases": getReleasesScheme,
  "/stats/code_frequency": getCodeFrequencyScheme,
  "/stats/participation": getParticipationScheme,
  "/stats/punch_card": getPunchCardScheme,
};

const getGithubStatistics = (id, repos) => {
  const headers = {
    "User-Agent": "Some user agent",
  };

  const requests = repos.map((repo) =>
    mapObjIndexed((handler, api) =>
      Future.tryP(() =>
        request
          .get({
            uri: `https://api.github.com/repos${
              url.parse(repo).pathname
            }${api}?client_id=${clientId}&client_secret=${clientSecret}`,
            headers,
          })
          /**
           * We should respect github rate limits (5000 request per hour), so
           * we check remaining requests count after every request and if it lower than
           * some threshold value, we sleep till specified limit reset time.
           */
          .on("response", updateLimitReset)
          .then(JSON.parse)
          .then(handler.bind(null, id, repo))
      )
    )(apis)
  );

  return Future.parallel(coinsParallelRequestsLimit, values(requests[0]));
};

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
