# 15: Image attachments and vision routing

**Phase 2 · Depends on: 14**

---

Read `docs/MEMORY.md` §6, `docs/ROUTING.md` §6.1 (vision auto-fill), and `docs/DECISIONS.md` D24.

1. Allow `.png`, `.jpg`, `.jpeg`, `.webp` and `.gif` uploads (≤ 10 MB, `kind='image'`, `text_status='none'`), in both the composer and the Files tab. Show thumbnails.
2. In `chat`: if any attachment is an image, set `requirements.vision = true`. For each image, create a signed URL valid for 10 minutes, and send it as OpenAI-format content parts (`{type:'image_url', image_url:{url}}`). Never store data URLs in `prompt_logs`; store `[image: name]` placeholders instead.
3. Manual or sticky models without vision → `error{code:'model_lacks_vision'}` with a UI prompt listing 3 vision models (free first).
4. Show an 👁 badge on the routing card when vision was required.

---

### Verify (don't paste)
- [ ] Phase 2 acceptance item 5. Run the whole Phase 2 checklist, and update DECISIONS.md if anything changed.
