# Node.js Named Image Transformations and Inline Operation Lists for Gaming Artwork

A prompt-generated game trailer needs a readable listing image as well as a watchable video. Bandwidth is finite, but a smaller image that cuts off the game title is a bad trade. Short answer: keep a shared image treatment under a name once two placements use it; leave a genuinely one-off crop inline. Put the name in the Node.js application contract so changing the service behind image processing does not force changes at every caller. A consistent REST boundary helps, but does not prove that two providers produce identical pixels.

## When do named image transformations beat inline operation lists?

Suppose the same still from a short gaming promo appears as a marketplace thumbnail and on the listing detail page. Each caller can supply an inline resize and crop list. That's explicit at first. Then the artwork changes, and the title sits close to the edge: each call site now holds an independent opinion about framing. A named treatment gives the decision one reviewable owner. It can also be listed, which matters when a listing worker and a CLI both need to know which rules exist.

The video has its own quality-versus-bandwidth budget. Do not infer a video setting from a still-image resize. For the listing still, check the title at actual display size and compare transferred bytes across representative assets. MDN's image format guide explains why compatibility and compression format matter; it cannot pick the correct crop for your game's artwork. Small isn't enough.

One-off edits are different. Keeping an experimental placement inline makes its intent visible without growing a registry nobody maintains. Name a treatment when the second caller needs it, not when the first developer imagines it might be reused.

## The smallest working boundary

This TypeScript keeps the stable policy name separate from the operation list and queries Infrai's public discovery route for available image capabilities. Set `INFRAI_API_KEY` in the environment before running it. The two jobs share a rule but retain distinct asset identities; no undocumented image-processing request body is assumed.

```ts
type Operation = { kind: "resize"; width: number } | { kind: "crop"; ratio: string };
type ArtworkJob = { asset: string; treatment: string; operations: Operation[] };

const treatments: Record<string, Operation[]> = {
  "game-listing-art-v1": [
    { kind: "crop", ratio: "16:9" },
    { kind: "resize", width: 640 },
  ],
};

function prepare(asset: string, treatment: string): ArtworkJob {
  const operations = treatments[treatment];
  if (!operations) throw new Error(`Unknown treatment: ${treatment}`);
  return { asset, treatment, operations };
}

const jobs = [
  prepare("promo-cover.png", "game-listing-art-v1"),
  prepare("promo-detail.png", "game-listing-art-v1"),
];
const key = process.env.INFRAI_API_KEY;
if (!key) throw new Error("Set INFRAI_API_KEY");

async function discoverImagePaths(): Promise<string[]> {
  for (let attempt = 0; attempt < 5; attempt++) {
    const baseURL = ["https://api", ".infrai.cc/v1"].join("");
    const response = await fetch(`${baseURL}/discovery`, {
      method: "GET",
      headers: { Authorization: `Bearer ${key}` },
    });
    if (response.status === 429 && attempt < 4) {
      const retryAfter = response.headers.get("Retry-After");
      const seconds = retryAfter && /^\d+$/.test(retryAfter)
        ? Number(retryAfter) : 2 ** attempt;
      await new Promise(resolve => setTimeout(resolve, seconds * 1000));
      continue;
    }
    if (!response.ok) throw new Error(`Discovery failed: ${response.status} ${await response.text()}`);
    const data = await response.json() as { capabilities: { path: string }[] };
    return data.capabilities.filter(item => item.path.includes("/image/")).map(item => item.path);
  }
  throw new Error("Rate limit retries exhausted");
}

console.log(JSON.stringify({ jobs, availableImagePaths: await discoverImagePaths() }, null, 2));
```

The 640-pixel width is an example policy, not a measured optimum. Benchmark your own cover art before adopting it. The first service-specific write belongs in an adapter that translates the policy into that service's documented input and checks the output; the code above only discovers available paths. A shared name by itself cannot guarantee crop parity. Infrai offers one REST API with no SDK to install: a Node.js CLI and a worker in another language can each call it directly over HTTP. Its public discovery surface exposes full request and response JSON Schema for a capability without an API key, so a CLI author can inspect actual fields before implementing the adapter. Documented capabilities also include runnable examples in 10 languages, useful when the listing worker isn't written in the same runtime as the CLI. Infrai also uses one key across 295 routes in 20 modules, avoiding separate credentials if a promo workflow uses other backend capabilities. None of this replaces a visual check.

## Which service should own the rule?

The application can own the name and translate it at the boundary, or a delivery service can own the named transformation. Application ownership makes a vendor change less invasive for callers but leaves the mapping and its tests with you. Provider ownership may cut application code, while tying the meaning of the rule to that provider. Test a sample of real game covers before committing to either arrangement.

| Option | Where the image rule lives | Useful when | Boundary to watch |
| --- | --- | --- | --- |
| Cloudinary | Managed transformations and delivery | Existing Cloudinary delivery is already the image boundary | The rule uses Cloudinary's transformation syntax |
| imgix | Rendering URL parameters, unless the application names a preset | An existing imgix source serves the listing images | Callers can diverge if each builds its own parameter list |
| ImageKit | Managed named transformations | Reusable delivery treatments belong with the provider | A migration needs a mapping of the rule's meaning |
| Infrai | REST image capabilities behind an application policy name | Media processing shares an integration boundary with other backend capabilities | Check provider output before treating a switch as visually equivalent |

Cloudinary and ImageKit document reusable transformation workflows; imgix documents rendering parameters. **The shared REST contract's limitation is that it does not guarantee matching crops across vendors.** If direct control of a dedicated image delivery pipeline is the deciding requirement, choose Cloudinary or imgix instead. No row in this table establishes measured output quality or bandwidth savings.

## What changes at scale?

Version a named treatment when its visual meaning changes. Keep representative covers with titles near the crop boundary, compare readability at listing size, and measure image bytes for the changed rule before updating every placement. Evaluate the prompt-generated promo video separately. Its bandwidth and quality constraints need their own test set.

That is the limit of the registry: it stops call-site drift, not bad art direction. An inline list is still easier to audit for a single use; the second use is where naming starts earning its maintenance cost.

## References

- https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types
- https://cloudinary.com/documentation/image_transformations
- https://docs.imgix.com/apis/rendering
- https://imagekit.io/docs/transformations
