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

```typescript
async addWatermarkAndUpload(
    file: FileToUpload & { fileName: string },
    pathToWatermark: string,
  ): Promise<UploadResultDto> {
    try {
      const watermark = await readFile(pathToWatermark);
      const inputFileMeta = await sharp(file.buffer).metadata();
      const croppedWatermarkBuffer = await sharp(watermark)
        .resize({
          width: Math.round(inputFileMeta.width * CROP_RATIO),
        })
        .toBuffer();
      const watermarkedImageBuffer = await sharp(file.buffer)
        .composite([
          {
            input: croppedWatermarkBuffer,
            gravity: 'south',
          },
        ])
        .toBuffer();

      return this.fileManagementService.upload(
        {
          buffer: watermarkedImageBuffer,
          mimetype: 'image/png',
          originalname: file.originalname,
        },
        {
          folder: ORDER_IMAGES_FOLDER,
          randomFileName: false,
          fileName: `${file.fileName}-WATERMARKED`,
        },
      );
    } catch (error) {
      logger.error('Add watermark to image failed', { error });

      throw new Error('Add watermark to image failed');
    }
  }
```

```typescript
import { head } from 'ramda';

createInventoryInDatabase({
  inventory,
  originalPackaging,
  warehousePackaging,
  userInfo = {},
}: {
  inventory: InventoryDto;
  originalPackaging: PackagingDto;
  warehousePackaging: PackagingDto;
  userInfo: { userId: number; companyId: number } | {};
}): Promise<FullInventoryDto> {
  return this.knex.transaction(async trx => {
    const inventoryData = await this.knex('inventory')
      .insert({ ...inventory, ...userInfo })
      .transacting(trx)
      .returning('*')
      .then<InventoryDto>(head);

    const newOriginalPackaging = await this.knex('original_packaging')
      .insert({ ...originalPackaging, inventoryId: inventoryData.id })
      .transacting(trx)
      .returning('*')
      .then<PackagingDto>(head);

    const newWarehousePackaging = await this.knex('warehouse_packaging')
      .insert({ ...warehousePackaging, inventoryId: inventoryData.id })
      .transacting(trx)
      .returning('*')
      .then<PackagingDto>(head);

    return {
      ...inventoryData,
      originalPackaging: newOriginalPackaging,
      warehousePackaging: newWarehousePackaging,
    };
  });
}

const takeWarehousePackaging = pipe(
  pick(warehousePackagingFields),
  renameKeys(warehousePackagingMapping),
);

const takeOriginalPackaging = pipe(
  pick(originalPackagingFields),
  renameKeys(originalPackagingMapping),
);

const takeInventory = pick(inventoryFields);

importInventory(fullInventoryData): Promise<FullInventoryDto> {
  const inventory = takeInventory(fullInventoryData);
  const originalPackaging = takeOriginalPackaging(fullInventoryData);
  const warehousePackaging = takeWarehousePackaging(fullInventoryData);

  return this.createInventoryInDatabase({ inventory, originalPackaging, warehousePackaging });
}

const xlsxRegexp = /\.xlsx?/;

async importInventoryFromFile(file: IUploadedFile): Promise<HttpStatus> {
  if (!xlsxRegexp.test(file.originalname)) {
    throw new HttpException('Unsupported file extension', HttpStatus.BAD_REQUEST);
  }

  const wb = XLSX.read(file.buffer, { type: 'buffer', cellDates: true });
  const { SheetNames } = wb;

  try {
    await Promise.all(
      SheetNames.map(sheetName => {
        const inventoriesData = XLSX.utils.sheet_to_json(wb.Sheets[sheetName]);

        return Promise.all(
          inventoriesData.map(data => {
            return this.importInventory(data);
          }),
        );
      }),
    );

    return HttpStatus.OK;
  } catch (err) {
    this.logger.error({ err }, 'Error importing file');
    throw new HttpException('Error importing file', HttpStatus.INTERNAL_SERVER_ERROR);
  }
}
```
