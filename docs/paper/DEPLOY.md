# Deploying the FlyRank paper site

1. Copy the contents of this folder into a `paper/` directory in your public GitHub repository.
2. Ensure `paper/index.html` and `paper/assets/` are committed.
3. In GitHub, open Settings → Pages.
4. Choose Deploy from a branch, select `main`, and select `/paper` as the folder if available.
   If the Pages UI does not offer `/paper`, publish from `/docs` or the repository root instead and place the files there.
5. Wait for the deployment workflow to finish, then open the published URL in a private/incognito window.
6. Update `submission/paper_url.txt` so it contains only the exact public URL, one line.
7. Check all image/CSV links and confirm no private data is published.

Review before publishing:
- Verify all reported metrics against the final notebook run.
- Add direct links to committed notebook files.
- Keep only synthetic page aliases in the public ranking CSV.
- Do not claim the full warehouse was used unless it was.
