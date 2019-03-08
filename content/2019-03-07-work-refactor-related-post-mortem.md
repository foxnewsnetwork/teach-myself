# Why was missing sku, productName, and tileTitle so hard to fix?

For context, in our [delegate-platform-search#sendClickTileEvent()](https://github.sie.sony.com/SIE/apollo-ps4/blob/a71aa7122bf96f77f37026de5406b3d231d636b6/app/components/delegate-platform-search/component.js#L329), we send out events with the following signature:

```javascript
sendClickTileEvent(game, parentTitle) {
  // setup...
  const specificPayload = {
    productName: tileTitle, // is null
    productId: tileId,
    sku, // is null
    catalogId,
    catalogTrackingId,
    tile: {
      tileType: 'product',
      tileId,
      tileTitle, // is null
      tilePosition, // is null (but don't care)
      tileTrackingId, // is null (but don't care)
      rowPosition, // is null (but don't care)
      parentTitle
    }
  };
  // ...dispatchEvent
}
```

But, as reported in [APOLLOPS-3686](https://jira.sie.sony.com/browse/APOLLOPS-3686), we would occasionally see the following fields as missing in our event payload

- `sku`
- `tile.tileTitle`
- `productName`

>sidenote: we also occasionally see other fields null, but we don't care about those right now

At first glance, this seems like an easy fix, let's find out what the structure of `game` looks like and just make sure we have the right fields where we need them!

Easy!

## Let's see what calls `sendClickTileEvent`

From our code, we have:

- [enterPressedSearchResult#L289](https://github.sie.sony.com/SIE/apollo-ps4/blob/a71aa7122bf96f77f37026de5406b3d231d636b6/app/components/delegate-platform-search/component.js#L289)
```javascript
enterPressedSearchResult(game) {
  this.sendClickTileEvent(game, TileTypesLabel.SEARCH_RESULT);
}
```
- [enterPressedViewedResult#L293](https://github.sie.sony.com/SIE/apollo-ps4/blob/a71aa7122bf96f77f37026de5406b3d231d636b6/app/components/delegate-platform-search/component.js#L293)
```javascript
enterPressedViewedResult(game) {
  this.sendClickTileEvent(game, TileTypesLabel.RECENTLY_VIEWED);
}
```
- [enterPressedTrendingSearch#L297](https://github.sie.sony.com/SIE/apollo-ps4/blob/a71aa7122bf96f77f37026de5406b3d231d636b6/app/components/delegate-platform-search/component.js#L297)
```javascript
enterPressedTrendingSearch(game) {
  this.sendClickTileEvent(game, TileTypesLabel.TRENDING_SEARCH);
}
```

Hmmm... ok, everything looks symmetric and sensible, so it must be a case that the `game` object referred to by each method is actually different from each other.

Surely, I can just find out what those types are and normalize them. Perhaps something like what I have on [this branch](https://github.sie.sony.com/SIE/apollo-ps4/blob/APOLLOPS-3686/super-click-missing-fields-attempt-2/app/components/delegate-platform-search/component.js#L294-L308) would work!

```typescript
enterPressedSearchResult(item: GridCanvasItem) {
  const game = gridCanvasToCarouselTile(item);
  console.log('enterPressedSearchResult(game)', game)
  this.sendClickTileEvent(game, TileTypesLabel.SEARCH_RESULT);
},

enterPressedViewedResult(game: CarouselTileItem) {
  console.log('enterPressedViewedResult(game)', game)
  this.sendClickTileEvent(game, TileTypesLabel.RECENTLY_VIEWED);
},

enterPressedTrendingSearch(item: GridCanvasItem) {
  const game = gridCanvasToCarouselTile(item);
  console.log('enterPressedTrendingSearch(game)', game)
  this.sendClickTileEvent(game, TileTypesLabel.TRENDING_SEARCH);
},
```

After a bit confirmatory testing, I get the following 2 types of tile objects (see [this file](https://github.sie.sony.com/SIE/apollo-ps4/blob/APOLLOPS-3686/super-click-missing-fields-attempt-2/app/components/delegate-platform-search/utils.js)):

```typescript
type GridCanvasItem = {
  id: string,
  item: {
    extended: {
      streamingSku: {
        id: string
      }
    },
    name: string,
  },
  itemIndex: number,
  row: number,
  rowIndex: number,
  x: number,
  y: number
};

type CarouselTileItem = {
  id: string,
  name: string,
  tileIndex: number,
  extended: {
    streamingSku: {
      id: string
    }
  }
};
```

Ok, so far so good, `GridCanvasItem` has slightly more information than `CarouselTileItem`, but luckily we can "down-convert" (plus certain fields we straight-up don't care about)

## Surely It'll work now, right?

Wrong, after testing, it seems the `enterPressedViewedResult` also on occasion gives a very unhelpful "just-id" type:

```typescript
type JustIdTile = {
  id: string
};

enterPressedViewedResult(tile: CarouselTileItem | JustIdTile): void;
```

With just and "id" string, there's no way I have enough information to populate the `sku` and `productName` fields

>"Well that's weird... how did the old search work with just an id field?" ~ me

Perhaps they got the details of `sku` and `productName` from the results of their `catalog` query.

But after an investigation into the old search reducer and delegate-search, they do the same workflow as platform-search. Notably:

```flow
st=>start: load search reducer
oploc=>operation: read recently visited from localStorage
opload=>operation: load platform-search
opread=>operation: populate via search.visitedPdps
opuser=>operation: user clicks on a tile
opsave=>operation: saveVisitedPdp({ id, name, extended })
opstore=>operation: store to localStorage

st->oploc->opload->opread->opuser->opsave->opstore
```

*AND* they do **NOT** get data from catalog - instead, data for recently viewed should just be correct. Lol well, that's ok, I guess, we don't have access to the `catalog` anyway in platform-search...

>In other words: there is no way to recover from have just an id with no additional info

But then, if we never expected 'em to be broken, just where in hell did these `JustTileId` type tiles come from?

## It was the grid-canvas switch search!

In [this pr](https://github.sie.sony.com/SIE/apollo-ps4/pull/2159/files#diff-561dd476ca1e136dcc214dd3ba8e2ef7R120) we rolled in `{{grid-canvas}}` in place of the previously non-performant `{{grid-focus}}` system. However, one thing that we overlooked was that the `enterPressed` action regurgitated the `grid-canvas` [tile type](https://github.sie.sony.com/SIE/psnow-utilities/blob/a23edadc077963b11a22ea6bad77d0340a2f9ad9/src/grid-canvas/index.ts#L95):


```typescript
export type highlightedGridItem = {
  id: string;
  row: number;
  rowIndex: number;
  itemIndex: number;
  item: gridItem;
  x: number;
  y: number;
};
```

which is *NOT* our previously expected `CarouselTileItem`.

Notably, when a developer performs

```javascript
const { id, name, extended } = gridItem;
// name === undefined
// extended === undefined
```

because they never existed on a `highlightedGridItem`!

However, because we pull `visitedPdp` data from `localStorage`, it was easy for QA to note that the events are wrong.

After all, if they clicked into a recently viewed tile, chances are it was the correct format from localStorage, and thus would *NOT* have revealed the problem!

## What this all means

1. We have a huge data consistency bug on our hands (see my example - note that games clicked from search results and trending searchs result in a `JustIdTile`)
```json
[
  {
    "id": "PSC001-BLKA00000_01-PSNW010000000000"
  },
  {
    "id": "PSC001-BLKB00000_02-PSNW010000000000"
  },
  {
    "id": "IV0003-NPXS29532_00-NOW0000000000000"
  },
  {
    "id": "PSC001-BLKD00000_02-PSNW010000000000"
  },
  {
    "id": "PSC001-BLKA00000_02-PSNW010000000000"
  },
  {
    "id": "UP9000-EXDG10203_00-SF00010000000000"
  },
  {
    "id": "PSC001-GKUX00001_00-PSNW010000000000",
    "name": "Apollo Queue Test with very long title because I DON'T KNOW?!!",
    "extended": {
      "streamingSku": {
        "id": "PSC001-GKUX00001_00-PSNW010000000000-U001"
      }
    }
  },
  {
    "id": "ES0009-TELL20170_00-MORE110000000000"
  }
]
```
2. *IF* we care to fix this bug, we *WILL* have to ship a release where we wipe the user's recently viewed
3. As a bonus, we get `tileTrackingId` and those other fields that should be there but aren't to show up for free
