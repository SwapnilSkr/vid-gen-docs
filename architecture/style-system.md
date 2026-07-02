# Per-Niche Style System

_Decisions from the animation-style research pass (2026-07-01). Sources: [OpusClip 17 styles](https://www.opus.pro/blog/viral-ai-video-styles-2026), [virvid styles guide](https://virvid.ai/blog/ai-video-styles-guide-2026), [scarystories horror](https://scarystories.live/blog/best-ai-horror-video-generators-2026), [OutlierKit faceless niches](https://outlierkit.com/resources/faceless-youtube-channels/), [SlashFilm liminal horror](https://www.slashfilm.com/2143744/what-is-liminal-horror-backrooms-trend-explained/)._

## What a "style" is (recap)
A style = **(image model + prompt suffix)** that produces the look + **(FFmpeg post + motion + caption skin)** that reinforces it. The image gen makes the look; FFmpeg only adds motion/mood. Implemented as `VisualStyle` entries inside a `NicheRecipe` (`server/src/config/niche-styles.ts`).

## The 2026 style landscape (what's actually trending)
- **Photorealistic cinematic** — "can't tell it's AI", drone/push-in, shallow DOF. Horror stories, movie-recap, sci-fi, motivation b-roll.
- **Analog / liminal horror** ⭐ — THE horror trend (Backrooms / Kane Pixels): low-res, VHS/camcorder noise, empty fluorescent liminal spaces, early-2000s digital artifacts. Distinct from cinematic horror.
- **Vintage / sepia archival** — aged photo, grain. Dark history, true crime. (Proven in our lab sample.)
- **2D motion graphics / Vox essay** ⭐ — charts, maps, lower-thirds, kinetic text. Finance + news. Research is explicit: finance wants **clean data-viz, NOT anime/sepia/cinematic**. Highest RPM ($15–30+).
- **Pixar-style 3D** — expressive 3D, subsurface scattering. Top converter; lighter facts, character explainers.
- **Ghibli / hand-painted 2D** — highest-*saving* format on Reels. Lo-fi loops, cozy/lifestyle.
- **Claymation / stop-motion** — thumbprints, 12–24fps stutter, Aardman. Brand-safe, high shareability; fun facts, comedy.
- **Anime / cel-shaded** — mythology, sci-fi lore, 18–26 demo.
- **Miniature diorama / tilt-shift** — emerging; isometric tiny worlds. Niche facts/travel.
- **AI lo-fi loops** — endless hypnotic autoplay scenes. Its own niche.
- **Brainrot edits / gameplay** — stacked splits + Subway-Surfers loop + kinetic captions. This IS the Reddit/AITA look (gameplay strategy, no AI images).

## Locked per-niche mapping

| Niche | Strategy | Style pool (rotates anti-slop) | Img tier | Hero | Caption skin |
|---|---|---|---|---|---|
| **reddit / aita** | gameplay_overlay | gameplay loop (no AI images) | — | never | bouncing word |
| **dark_history** | image_kenburns | vintage_sepia | cheap | never | serif lowerthird |
| **true_crime** | image_kenburns | noir_reconstruction, vintage_sepia | cheap | trend | serif lowerthird |
| **facts** | image_kenburns | clean_editorial, pixar_3d | cheap | never | word_pop |
| **fun_facts** | image_kenburns | claymation, pixar_3d | value | never | word_pop |
| **motivation** | image_kenburns | photoreal_cinematic | cheap | never | bold_center |
| **stoicism** | image_kenburns | marble_classical | cheap | never | serif elegant |
| **finance** | motion_graphics* | flat_mograph / data_viz | cheap | never | kinetic_highlight |
| **news** | motion_graphics* | mograph_map / editorial | cheap | never | lowerthird |
| **science_space** | image_kenburns | hyperreal_cosmic | value | trend | clean |
| **horror** | hybrid_scene | analog_liminal ⭐, photoreal_dark | value | one_climax | word (whisper) |
| **mythology** | hybrid_scene | painterly_epic, anime_cel | value | one_climax | epic_titlecard |
| **movie_recap** | hybrid_scene | photoreal_cinematic, surreal_whatif | value | one_climax | minimal |
| **lofi** | image_kenburns (loop) | ghibli_cozy, painterly_night | cheap | never | none |

\* `motion_graphics` strategy is not built yet → finance/news temporarily run on `image_kenburns` with the `clean_editorial`/`flat_mograph` style until the mograph renderer lands.

## Decisions locked
1. **Style rotation per video, consistent within a video.** Each niche has a *pool*; a reel picks ONE style (rotated) so a channel's videos vary (anti-slop) but each video is internally coherent.
2. **Finance/news are motion-graphics, not pretty stills** — highest RPM, and the audience expects charts/data. Flagged as the reason to build the `motion_graphics` strategy next after Reddit.
3. **Horror leads with analog/liminal**, not generic cinematic — it's the current viral aesthetic.
4. **Image tier per niche:** most niches are fine on `cheap` stills; horror/mythology/science/fun_facts get `value` for the richer look; hero-video niches gate one clip.
5. **Trending styles kept as first-class options** (claymation, pixar_3d, ghibli, anime_cel, diorama) so we can chase a trend by adding a style to a pool — no code change, just a registry entry.
