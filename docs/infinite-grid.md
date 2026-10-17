# Infinite tattoo exploration prototype

This page ports the **viewport behaviour** of `components/DiscoveryGridClient.tsx` in `magickmomo/youwontwearthistshirt` into this SvelteKit app without importing the React component, Supabase discovery state, credit purchases, or Shopify.

Visit `/explore` to pan an unbounded integer-coordinate tile grid, zoom at the pointer, inspect and download a design.

Each visible cell draws a small deterministic SVG motif locally using its coordinate hash. **These are temporary test graphics, not AI-generated or curated tattoo designs.** Replace the `motif(cell)` function with a real artwork source after establishing visual quality. The coordinate system, pan/zoom behaviour, visible-cell window and selection experience remain reusable.

The existing homepage mystery-roll/Stripe flow remains unchanged. No purchases or API credentials are required for this prototype.

Run `npm install`, `npm run check`, and `npm run build` locally before merging.