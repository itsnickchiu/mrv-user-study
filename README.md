# mrv-user-study

Pairwise video preference study. Built from the lab's `revp-user-study` template (Vue 3 + Vite, deployed to
GitHub Pages by `.github/workflows/deploy.yml`). The home page text (consent form, optional lottery e-mail,
IRB information) is copied from the template. Only the title and the procedure paragraph (24 video pairs,
about 10 minutes) differ.

## What a participant does
24 pairs, about 10 minutes.
- 16 comparison pairs on 16 different scenes, with two questions each (A or B).
- 2 attention checks, mixed into the comparison pairs.
- 6 problem pairs. A one-sentence problem is shown, with two questions each (A, B or about the same).

Left and right are randomized per pair. Both videos must be watched to the end before the answers unlock.

## Collecting answers (Google Form, same as the template)
1. Create a Google Form with **10 short-answer questions**, in this order:
   `email, session, index, pair, left, right, answer1, answer2, seconds, lang`.
2. Use "Get pre-filled link" to read the `entry.<number>` id of each question.
3. Put the form's `.../formResponse` URL and the 10 entry ids, in the same order, into `.env`:
   ```
   VITE_GOOGLE_FORM_URL=https://docs.google.com/forms/d/e/<form id>/formResponse
   VITE_GOOGLE_FORM_ENTRIES=["entry.1","entry.2",...]
   ```
4. Link the form to a Google Sheet. Each submitted pair becomes one row.

Answer codes: `L` = left video, `R` = right video, `S` = about the same (problem pairs only).
The mapping from pair and video ids to methods is kept offline and is not part of this repository.

## Develop and deploy
```sh
yarn
yarn dev          # local preview
yarn build-pages  # what the GitHub Action runs
```
Push to `main` to deploy. In the repository settings, Pages must serve the `gh-pages` branch.
