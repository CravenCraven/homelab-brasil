# Runbook: changing the Brasil demo page

How I rebuilt projectpattie.com/brasil/demo from Mockup v5 on Nov 20 and 21, 2026, and how to change it again without the same surprises. Every command here is one I ran. Run them in `~/Developer/projectpattie`.

| Thing | Value |
| --- | --- |
| Repo | `projectpattie` (the page is part of the website, so it lives here) |
| The page | `src/pages/brasil/demo.astro` |
| Tile files | `src/content/demo/05-musica.md` to `09-ollama.md` |
| Roadmap data | `src/data/roadmap.json` |
| Local preview | `localhost:4321/brasil/demo` |
| Goes live through | Cloudflare Pages, after a merge to `main` |
| Branch I used | `brasil-demo-update` |

## Part 1: start on a branch

```
cd ~/Developer/projectpattie
git switch main
git pull
git switch -c brasil-demo-update
git branch --show-current
```

I should see: `brasil-demo-update`.

`-c` makes the branch. Without `-c`, it only moves to one that already exists.

## Part 2: run it next to the mockup

```
npm run dev
```

Open `localhost:4321/brasil/demo` with the mockup beside it. Go down both, section by section. Ctrl+C to stop.

## Part 3: find the spot before editing

Find lines by what they say, with line numbers:

```
grep -n 'Study streak\|class="streak"\|<footer' src/pages/brasil/demo.astro
```

The single quotes outside let the search hold double quotes inside.

Print just a range of lines:

```
sed -n '735,760p' src/pages/brasil/demo.astro
```

Open the file with the cursor already on a line:

```
nvim +411 src/pages/brasil/demo.astro
```

## Part 4: the edits that needed care

Before any Neovim command: press Esc first.

**Delete one line by what it says.** Type the whole pattern, with the words in it:

```
:g/<li>This portal<span>/d
```

Never stop at `:g/<li>/d`. That deletes every list item in the file. If something wrong gets deleted, press `u` before saving.

Check it from the terminal:

```
grep -n "This portal" src/pages/brasil/demo.astro
```

I should see: nothing.

**Change one thing on the current line:**

```
:s/class="streak"/class="streak" style="margin-top:0"/
```

Check:

```
grep -n 'margin-top:0' src/pages/brasil/demo.astro
```

**Add a line with sed.** `sed -i` prints nothing when it works. Do not run it a second time to be sure. I did, and got the button twice. Check with grep instead:

```
grep -n realbtn src/pages/brasil/demo.astro
```

I should see: one line. Two lines means there is an extra to delete.

## Part 5: things a script adds need `:global`

The study streak showed the 9 and no bars. The bars were there with no size and no color, because Astro only styles what is written in the page itself, and the bars are made by a script.

How I looked:

```
grep -n "bars" src/pages/brasil/demo.astro
```

I should see: the `.bars` style, the empty `id="bars"` box, and the script line that fills it. Then the browser console (Ctrl+Shift+J, reload): nothing red.

The fix:

```
sed -i 's|^\(\s*\)\.bar{|\1:global(.bar){|; s|^\(\s*\)\.bar\.off{|\1:global(.bar.off){|' src/pages/brasil/demo.astro
```

That turns `.bar{` into `:global(.bar){` and `.bar.off{` into `:global(.bar.off){`. Reload. The bars are back.

## Part 6: make sure the Música tile still opens the app

```
grep -n "slugOf(t)" src/pages/brasil/demo.astro
```

I should see: the tile links still use `` t.data.href ?? `/brasil/${slugOf(t)}` ``. If the `t.data.href ??` part is gone, the Música tile goes back to its write-up page.

In the preview: click Música (should go to the Cloudflare login), then click one other tile (should open its write-up).

## Part 7: commit and push

```
git add src/pages/brasil/demo.astro src/content/demo/ src/data/roadmap.json
git status --short
```

I should see: only those files.

```
git commit -m "feat(brasil-demo): update demo page to v5 mockup" -m "Three nodes, roadmap from the tracker, Try it previews, stack by layer, things that broke, hire-me banner, tile colors, streak bars fix."
```

The first `-m` is the title. The second is the details under it.

```
git push -u origin brasil-demo-update
```

Plain `git push` on a new branch fails with:

```
fatal: The current branch brasil-demo-update has no upstream branch.
```

That only means GitHub has not seen the branch yet. `-u origin <branch>` fixes it, once.

If a later `git push` says "Everything up-to-date" and I am not sure it went:

```
git log --oneline -1
```

I should see: my commit message.

## Part 8: pull request and merge

On GitHub, in the projectpattie repo:

1. Yellow bar: "brasil-demo-update had recent pushes." Click Compare & pull request.
2. Check it says base `main`, compare `brasil-demo-update`.
3. Create pull request.
4. Merge pull request, then Confirm merge.

## Part 9: check it live

Wait a few minutes. Cloudflare Pages builds the site after the merge. Then open projectpattie.com/brasil/demo with a hard refresh (Ctrl+Shift+R, or Cmd+Shift+R on the Mac).

If it still looks old, look at the Deployments list for the projectpattie project in Cloudflare. It says if the build is running, finished or failed. Do not look in GitHub Actions. The site does not deploy from there.

## When it breaks

| What I see | What it was |
| --- | --- |
| Something a script draws is missing or invisible, console is clean | Scoped styles. Wrap the style in `:global()`. |
| The same thing shows up twice on the page | A `sed -i` insert ran twice. `grep -n`, delete the extra line. |
| A lot of lines vanished in Neovim | A `:g/.../d` with too short a pattern. `u` to undo, before saving. |
| `no upstream branch` | First push of a new branch. `git push -u origin <branch>`. |
| Merged, live page looks the same | Pages still building, or the browser saved the old page. Wait, hard refresh. |
| Música tile opens the write-up page again | The `t.data.href ??` part of the tile link got lost. See Part 6. |

## What went in on Nov 21

Commit `294aeee`, merged as pull request #1 (`00480dc`). Seven files.

- Tile colors in the five tile files
- Three nodes everywhere, links to `homelab-brasil`
- Stack cards: Sealed Secrets, coach and "This portal" out. Coach and Live portal under Next.
- Study streak moved to the bottom, labeled sample data
- New sections: Try it, How it deploys, Stack by layer, Things that broke, Roadmap, hire-me banner
- Roadmap reads `src/data/roadmap.json` (199 items, 14 done, from the Brasil Roadmap Tracker as of Nov 7)

## Not done

- Too much orange
- Node cards: third card, labels, "sample numbers" heading
- Roadmap capstone wording, and a capstone line for the newest phase

Done since (Nov 24): SOPS + age moved to the Platform card and marked done in the table, and the Música demo login is on the tile and in Try it.
