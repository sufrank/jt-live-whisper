# 同步流程

這個 repository 使用兩個 remote：

- `origin`：你的 repository（`https://github.com/sufrank/jt-live-whisper.git`）
- `upstream`：原始 repository（`https://github.com/jasoncheng7115/jt-live-whisper.git`）

## 標準同步流程

用這組指令從原始 repository 拉更新，並推送到你自己的 repository：

```bash
cd D:\Projects\Private\jt-live-whisper
git remote -v
git fetch upstream
git checkout main
git merge upstream/main
git push origin main
```

## 合併前先查看更新

如果你想先確認有哪些新提交，再決定是否合併：

```bash
git fetch upstream
git log --oneline main..upstream/main
```

## 另一種做法：rebase

如果你想保留較線性的提交歷史，而不是產生 merge commit：

```bash
git fetch upstream
git checkout main
git rebase upstream/main
git push origin main
```

`rebase` 會改寫本地提交歷史。如果你想用比較穩定、保守的方式，建議使用 `merge`。

## 發生衝突時怎麼處理

如果在 `git merge upstream/main` 或 `git rebase upstream/main` 時發生衝突，可以用以下流程處理：

```bash
git status
```

先查看哪些檔案有衝突，Git 會標示需要處理的檔案。

打開衝突檔案後，你會看到類似下面的內容：

```text
<<<<<<< HEAD
你的內容
=======
upstream 的內容
>>>>>>> upstream/main
```

請手動修改成你要保留的最終內容，並刪除這些衝突標記。

修改完成後：

```bash
git add <檔案名稱>
```

如果你使用的是 `merge`，接著執行：

```bash
git commit
git push origin main
```

如果你使用的是 `rebase`，接著執行：

```bash
git rebase --continue
git push origin main
```

如果你想放棄這次衝突處理：

```bash
git merge --abort
```

或是如果你當時使用的是 `rebase`：

```bash
git rebase --abort
```

不確定怎麼合併時，先執行 `git status` 與 `git diff` 看清楚差異，再決定保留哪一邊的內容。
