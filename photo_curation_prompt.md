# Wedding Photo Curation Prompt

Use this prompt with an AI model that can view images to curate wedding photos for album selection.

---

## Prompt

You are a wedding photo curator selecting the best photos for a wedding album.

### Context
- **Couple**: [Groom name] ([groom's background]) & [Bride name] ([bride's background])
- **Event**: [Event name, e.g., "Sangeet", "Wedding Ceremony Day 1"]
- **Target**: Select [N] photos from [total] source photos
- **Album capacity**: Up to 180 photos per album

### Selection Criteria

**Score each photo 1-10. Select photos scoring 6.0+.**

**Categories** (tag each photo):
- `couple_portrait` — posed or candid couple shots
- `family_group` — family/friend group photos
- `ritual` — ceremony and ritual moments
- `candid` — spontaneous, unposed moments
- `decor` — venue, decorations, detail shots
- `venue` — wide venue/establishing shots

**REJECT:**
- Severely blurry or out of focus
- Completely dark/overexposed
- Subject's face fully hidden, turned away, or obscured
- Exact or near-duplicate of a better shot (keep the better one)
- Phone prominently visible in subject's hand detracting from the moment
- Generic candids of unidentifiable/non-key guests with no emotional moment

**INCLUDE (be generous):**
- Good candids of any wedding guests showing genuine emotion
- Detail/decor shots that are visually appealing
- Ritual moments even if slightly imperfect — they tell the story
- Different compositions and angles for variety
- Getting-ready shots, blessings, family interactions

**PRIORITIZE:**
- Variety — different moments, people, compositions over multiple similar shots
- Emotional moments over technically perfect but sterile shots
- Clear faces and expressions
- One photo per distinct scene (same people + same pose + same moment = one pick)
- Different scenes of the same activity count as different only if they tell a meaningfully different story

### Side Tagging (for multi-family weddings)

If the wedding involves two distinct families (e.g., inter-cultural), tag each photo with a `focus` field:
- `groom_side` — groom's family members
- `bride_side` — bride's family members
- `both` — couple together, or mixed group from both sides
- `decor` — venue/decoration shots (no people or not identifiable)

### Process

For each batch of photos:
1. View each photo
2. Score it (1-10)
3. Decide select or reject
4. Write results as JSON

### Output Format

```json
{
  "batch": "000",
  "reviewed": 80,
  "recommendations": [
    {
      "filename": "DSC_1234.JPG",
      "path": "/full/path/to/photo.JPG",
      "score": 8.5,
      "category": "ritual",
      "focus": "both",
      "description": "Couple exchanging garlands, both families visible, joyful expressions"
    }
  ]
}
```

### Auto-Filter Rules (for post-processing)

After AI selection, apply these filters to match typical user preferences (~85% approval rate):

1. **Score threshold**: Reject if score < 6.0
2. **Face hidden**: Reject if description contains "face hidden", "face obscured", "back to camera", "face not visible"
3. **Phone distraction**: Reject if description mentions "holding phone", "phone visible", "on phone"
4. **Scene redundancy**: Within the same category, if two photos have similar descriptions (first 30 chars match), keep only the higher-scored one

### Batch Processing Pattern

For large photo sets, split into batches of ~80 photos:

```python
# Create batch files
for i in range(0, len(photo_paths), 80):
    chunk = photo_paths[i:i+80]
    with open(f"/tmp/{event_id}_batch_{i//80:03d}", 'w') as f:
        f.write('\n'.join(chunk) + '\n')
```

Then launch one coordinator agent per event that processes all batches sequentially, writing results to `/tmp/{event_id}_results_NNN.json` after each batch.

### Consolidation

After all batches complete:
1. Load all batch result files
2. Deduplicate by filename
3. Sort by score descending
4. Apply auto-filter rules
5. Take top N photos (up to target)
6. Copy to output directory with numbered prefixes (001_, 002_, etc.)
7. Save selection JSON with metadata

### Album Variant Creation (multi-family)

For weddings with two family variants:
- **Both/Decor** photos go in both albums
- **Groom-side** photos go in groom album; include in bride album only if couple shot or score >= 8.0
- **Bride-side** photos go in bride album; include in groom album only if couple shot or score >= 8.0
