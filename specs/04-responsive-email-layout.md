# SPEC 04 — Responsive layout for the news and gospel emails

> **Status:** Implemented
> **Depends on:** SPEC 01, SPEC 02
> **Date:** 2026-10-14
> **Objective:** Change the HTML of the News Digest and Daily Gospel emails so the single column grows to 760px on a laptop, stays full width with less side padding on a phone, and keeps a fixed 760px width in Outlook desktop.

## Why this spec exists

Both emails are capped at `max-width:600px`.
On a laptop the column fills less than half of the screen, with wide empty margins.
On a phone the column fills the screen, but the outer and inner padding take space from the text.

The user chose a wider single column over a two-column layout.
The content, the order of the sections and the font sizes do not change.

This spec modifies the `Render Email` node of SPEC 01 and of SPEC 02.
The `Render Email` node of the Daily Gospel already includes the apologetic note of SPEC 03, and that part is not changed.

## Scope

**In:**

- The `Render Email` Code node in `projects/news-digest/workflows/news-digest.json`.
- The `Render Email` Code node in `projects/daily-gospel/workflows/daily-gospel.json`.
- Fluid-hybrid layout: the content table has `width="100%"` and `max-width:760px`. It fills a narrow screen and stops at 760px on a wide one.
- An Outlook-only "ghost table" of fixed width 760px around the content table, inside `<!--[if mso]>` conditional comments.
- A `<style>` block in `<head>` with one `@media (max-width:480px)` rule that reduces the side padding on phones.
- Two class names, `px-outer` and `px-inner`, on the cells whose side padding the media rule changes.
- All current inline styles stay as they are, so a client that drops `<style>` shows the 760px layout with today's padding.
- Updated comments in both nodes, which now say that `<style>` holds only the phone padding rule.
- Re-export of both workflows to their JSON files.

**Out of scope (for future specs):**

- A two-column layout on wide screens.
- English and Spanish blocks side by side for `language` = `both`.
- Larger font sizes on a laptop.
- Any change to font sizes, colours, labels, section order or content.
- Dark mode styles.
- Testing in a real Outlook desktop client.
- A "view in browser" link or a web version of the email.

## Data model

This feature introduces no new data structures. It reuses the models of SPEC 01, SPEC 02 and SPEC 03, and the output fields of both `Render Email` nodes are unchanged.

**Layout constants** — at the top of each `Render Email` node:

```js
const WIDTH = 760; // max width of the column, and fixed width in Outlook
```

**Phone padding rule** — one `<style>` block per email, in `<head>`:

```css
@media (max-width:480px) {
  .px-outer { padding-left:4px !important; padding-right:4px !important; }
  .px-inner { padding-left:<inner>px !important; padding-right:<inner>px !important; }
}
```

| Email | Cell | Class | Side padding today | Side padding on phone |
| --- | --- | --- | --- | --- |
| Daily Gospel | Outer cell around the column | `px-outer` | 10px | 4px |
| Daily Gospel | Block header, each section, footer | `px-inner` | 28px (footer 24px) | 18px |
| News Digest | Outer cell around the column | `px-outer` | 12px | 4px |
| News Digest | Each article card, vocabulary box | `px-inner` | 20px | 14px |

**Outlook ghost table** — around the content table, in both emails:

```html
<!--[if mso]><table role="presentation" width="760" align="center" cellpadding="0" cellspacing="0"><tr><td><![endif]-->
  ... content table with width="100%" and max-width:760px ...
<!--[if mso]></td></tr></table><![endif]-->
```

Conventions:

- The media rule changes only `padding-left` and `padding-right`. Top and bottom padding are the same on every screen.
- The rule uses `!important` because the inline styles would win otherwise.
- The News Digest header row and footer row keep their 4px side padding and get no class.
- The Daily Gospel action box (`Today` / `Hoy`) keeps its 18px side padding and gets no class.
- `WIDTH` is used in the content table's `max-width` and in the ghost table's `width`, so the two cannot differ.
- The `<meta name="viewport">` tag that already exists in both emails stays.

## Implementation plan

1. In the Daily Gospel `Render Email` node, add `WIDTH`, change `max-width:600px` to `max-width:${WIDTH}px` and wrap the content table in the ghost table. Import with `docker exec n8n n8n import:workflow --input=<file>`. Check: copying the output HTML into a browser at 1280px wide shows a 760px column, and at 375px wide shows the column at full width with today's padding.
2. In the same node, add the `<style>` block in `<head>` and the classes `px-outer` and `px-inner` on the cells listed in the Data model. Update the node comment. Check: in the browser at 375px wide the side padding is 4px outside and 18px inside, and at 1280px it is unchanged.
3. Run the Daily Gospel workflow manually with the starter row (`both`) and check the received email in Gmail web, the Gmail mobile app and Apple Mail. Re-export the workflow to `projects/daily-gospel/workflows/daily-gospel.json`.
4. In the News Digest `Render Email` node, add `WIDTH`, change `max-width:600px` to `max-width:${WIDTH}px` and wrap the content table in the ghost table. Import the workflow. Check: in the browser the column is 760px at 1280px wide and full width at 375px.
5. In the same node, add the `<style>` block and the classes `px-outer` and `px-inner` on the cells listed in the Data model. Update the node comment. Check: in the browser at 375px wide the side padding is 4px outside and 14px inside each card.
6. Run the News Digest workflow manually and check the received email in Gmail web, the Gmail mobile app and Apple Mail. Re-export the workflow to `projects/news-digest/workflows/news-digest.json`.

## Acceptance criteria

- [x] `docker exec n8n n8n import:workflow --input=<file>` imports both workflow files without errors.
- [x] Neither `Render Email` node contains the string `max-width:600px`.
- [x] Both `Render Email` nodes define `WIDTH = 760` and use it for the `max-width` and the ghost table `width`.
- [x] In Gmail web on a laptop, with the browser window wider than 1000px, the content column of both emails is 760px wide and centred.
- [x] In Gmail web, with the browser window narrower than 760px, the content column of both emails fills the available width with no horizontal scroll.
- [x] In the Gmail mobile app, both emails show one column with no horizontal scroll.
- [x] In the Gmail mobile app, the Daily Gospel text starts closer to the screen edge than before this spec.
- [x] In Apple Mail on iPhone, both emails show one column with no horizontal scroll and the reduced side padding.
- [x] In Apple Mail on macOS, with a window wider than 1000px, the content column of both emails is 760px wide.
- [ ] The HTML of both emails contains a `<!--[if mso]>` table with `width="760"` opening before the content table and closing after it.
- [x] The HTML of both emails contains exactly one `<style>` block, and it holds only the `@media (max-width:480px)` rule.
- [x] With the `<style>` block removed from the HTML, a browser at 375px wide still shows the full email in one column with no horizontal scroll.
- [x] The font sizes, colours, labels and order of the sections are the same as before this spec in both emails.
- [x] The Daily Gospel email for a `both` row still shows the English block first and the Spanish block below.
- [x] The `subject`, `html` and recipient fields of both `Render Email` outputs keep the same names as before this spec.

## Decisions

- **Yes:** both emails in one spec. Chosen by the user. The change is the same pattern in two nodes.
- **Yes:** a wider single column on a laptop. Chosen by the user over two columns and over English and Spanish side by side.
- **No:** two columns on a laptop. More layout code, and the gospel reads from top to bottom.
- **No:** English and Spanish side by side. It only helps `both` rows.
- **Yes:** 760px as the new maximum width. Chosen by the user. The 19px gospel text stays at about 75 to 80 characters per line.
- **No:** 800px or 900px. Lines of the gospel text get too long to follow.
- **Yes:** fluid-hybrid layout as the base. Chosen by the user. It works without `<style>`, so it survives clients that strip it.
- **No:** a layout based on media queries. Clients that drop `<style>` would fall back to a layout the user did not choose.
- **Yes:** one small `@media` rule only for the phone padding. Chosen by the user. Fluid-hybrid alone cannot change padding by screen size, and a client that drops the rule only keeps today's padding.
- **No:** smaller padding on every screen. Not selected.
- **Yes:** phone padding of 4px outside and 18px inside for the gospel, 4px and 14px for the news. Chosen by the user. About 30px more text width on a 390px phone.
- **No:** 0px outer padding. The sheet border would touch the screen edge.
- **Yes:** today's font sizes on every screen. Chosen by the user. No font rule in `<style>`.
- **Yes:** an Outlook ghost table with fixed width. Outlook desktop ignores `max-width` and would stretch the email to the full window.
- **Yes:** Outlook supported without a real client test. The user has no Outlook desktop. The criterion checks the HTML instead.
- **Yes:** the clients to check are Gmail web on a laptop, the Gmail mobile app and Apple Mail. Chosen by the user, with Outlook checked in the HTML only.
- **Yes:** one constant `WIDTH` per node. The column width and the Outlook width cannot drift apart.
- **No:** a shared file or node for the layout. The two workflows stay independent, as in SPEC 02.
- **Yes:** a new spec that depends on SPEC 01 and SPEC 02, which stay as `Implemented`. The history of what was built first is kept.

## Risks

| Risk | Mitigation |
| --- | --- |
| Gmail drops the whole `<style>` block when it finds a rule it does not accept. | The block holds a single `@media` rule with plain properties. Without it the email shows today's padding, which is the SPEC 01 and SPEC 02 layout. |
| Gmail with a non-Google account in the Gmail app ignores `<style>`. | Same fallback: the column fills the screen with today's padding. |
| The ghost table is never seen in a real Outlook client. | The pattern is standard and only Outlook reads it. Other clients treat it as a comment. |
| A wider column makes long gospel lines harder to read on a laptop. | 760px keeps about 75 to 80 characters per line at the current 19px size. |
| A class name collides with a class used by the mail client. | The `px-` prefix is specific to these emails. |
| Gmail clips messages larger than 102KB, and the new markup adds bytes. | The ghost table, the `<style>` block and the classes add well under 1KB. |

## What is **not** in this spec

- Two-column layout.
- English and Spanish side by side.
- Larger fonts on a laptop.
- Changes to colours, labels, content or section order.
- Dark mode.
- Real Outlook desktop testing.
- "View in browser" link or web version.

Each one of those, if it lands, goes in its own spec.
