# Horizon update playbook (baseline-first, streamlined)

Update Shopify Horizon by inserting the vendor version immediately after the 2.1.1 baseline commit, then replay customizations on top.

## Variables
```bash
VERSION=2.1.4
VERSION_DIR="/tmp/horizon-$VERSION"
```

## 0) Prep
```bash
git fetch --prune
git status -sb

# Sanity check vendor folder
ls -1 "$VERSION_DIR" | head
[ -d "$VERSION_DIR/assets" ] || echo "Warning: expected assets/ in $VERSION_DIR"
```

## 1) Branch from 2.1.1 baseline
```bash
# Find the baseline commit
BASE=$(git log --grep="^Update Horizon theme" --pretty=format:%H -n 1)
echo "$BASE"
[ -n "$BASE" ] || echo "No Horizon update commit found; set BASE to the desired vendor update commit hash manually."

# Create branch at that baseline
git checkout -b "rebase/horizon-$VERSION" "$BASE"
```

## 2) Replace theme directories with vendor $VERSION
```bash
# Shopify theme directories to replace in full
THEME_DIRS="assets blocks config layout locales sections snippets templates"
for d in $THEME_DIRS; do
  rm -rf "./$d"
  if [ -d "$VERSION_DIR/$d" ]; then
    mkdir -p "./$d"
    cp -a "$VERSION_DIR/$d/." "./$d/"
  fi
done

git add -A
git commit -m "Update Horizon theme to $VERSION"
```

## 3) Replay your customizations on top
```bash
# Preview the commits that will be replayed
git --no-pager log --oneline "$BASE"..main

# Cherry-pick your changes after baseline onto the new vendor version
# Resolve conflicts if they arise (add files, then --continue)
git cherry-pick --reverse "$BASE"..main || true
# If conflicts appear, merge changes and fix files then:
#   - git add -A
#   - git cherry-pick --continue
# Else abort if necessary:
#   - git cherry-pick --abort
```

## 4) Push and open PR
```bash
git push -u origin "rebase/horizon-$VERSION"
```
Open a PR to merge into main and review the diff.

## 5) Merge, tag, and clean up
After merging the PR into main:
```bash
git checkout main
git pull --ff-only

git tag -a "vendor/horizon-$VERSION" -m "Horizon $VERSION vendor baseline"

git push origin "vendor/horizon-$VERSION"
```

## Rollback
```bash
# Abandon the update branch entirely
git checkout main
git branch -D "rebase/horizon-$VERSION"

# Undo last commit on the update branch (if needed)
# git reset --hard HEAD~1
```

## Notes
- This flow inserts a clean vendor baseline before your customizations, so no exclude/include syncing is required.
- Only Shopify theme directories are replaced; non-theme files (docs, scripts) are left intact.
- If the grep for 2.1.1 fails, locate the baseline commit hash manually (via git log) and set BASE to that hash.
