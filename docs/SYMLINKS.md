# Symlink Playlist Support

## Overview

Create Plex playlists from folders containing **symlinks** to music files, avoiding file duplication.

## Why Use Symlinks?

**Problem:** You want the same track in multiple playlists, but don't want duplicate files.

**Solution:** Use symlinks! Keep one copy of each file, create multiple playlist folders with symlinks.

```
Library Structure:
/media/music/
  Artist-A/track1.mp3  (real file, 5MB)
  Artist-B/track2.mp3  (real file, 4MB)

Playlist Structure:
/playlists/
  Favorites/
    track1.mp3 → /media/music/Artist-A/track1.mp3  (symlink, 0 bytes)
    track2.mp3 → /media/music/Artist-B/track2.mp3  (symlink, 0 bytes)
  
  Workout/
    track1.mp3 → /media/music/Artist-A/track1.mp3  (same symlink)
```

**Result:** 9MB of music, infinite playlists! 🎵

---

## How It Works

### Standard Matching (Old Behavior)
- Check if track path **contains** playlist folder path
- Works when folder structure matches Plex library

### Symlink Matching (New Behavior)
- **Scan** playlist folder for files
- **Resolve** each symlink to real file path
- **Match** real paths exactly against Plex database
- Falls back to standard matching if symlinks not detected

---

## Usage

### Creating a Symlink Playlist

#### 1. Create playlist folder with symlinks:

**Linux/macOS:**
```bash
# Create folder
mkdir ~/playlists/My-Favorites

# Add symlinks to your favorite tracks
ln -s "/media/music/Artist/Song1.mp3" ~/playlists/My-Favorites/
ln -s "/media/music/Artist/Song2.mp3" ~/playlists/My-Favorites/
ln -s "/media/music/Other/Song3.mp3" ~/playlists/My-Favorites/
```

**Windows (PowerShell - Run as Administrator or enable Developer Mode):**
```powershell
# Create folder
New-Item -ItemType Directory -Path "$HOME\playlists\My-Favorites"

# Create symlinks (requires admin or Developer Mode)
New-Item -ItemType SymbolicLink -Path "$HOME\playlists\My-Favorites\Song1.mp3" -Target "D:\music\Artist\Song1.mp3"
New-Item -ItemType SymbolicLink -Path "$HOME\playlists\My-Favorites\Song2.mp3" -Target "D:\music\Artist\Song2.mp3"
```

**Windows Alternative (Hard links - no admin needed):**
```powershell
# Hard links work without admin but only for files on same drive
New-Item -ItemType HardLink -Path "$HOME\playlists\My-Favorites\Song1.mp3" -Target "D:\music\Artist\Song1.mp3"
```

**Tip (Linux/macOS):** Use wildcards for bulk linking:
```bash
# Link all files from a folder
ln -s /media/music/Artist/*.mp3 ~/playlists/My-Favorites/

# Link specific pattern
ln -s /media/music/**/favorite*.mp3 ~/playlists/Best/
```

#### 2. Create playlist in app:

1. Launch Plex Folder Playlist Creator
2. Select folder: `~/playlists/My-Favorites`
3. Enter playlist name: "My Favorites"
4. Select library: "Music"
5. Click **Create Playlist**

#### 3. Verify in console (optional):

Enable console logging in settings, look for:
```
[findPlaylistTracks] No matches with standard path matching
[findPlaylistTracksBySymlinks] Scanning folder for symlinked tracks
[pathUtils] Found 3 audio files in folder
[findPlaylistTracksBySymlinks] Matched 3/3 tracks
Creating playlist: "My Favorites" with 3 items
```

---

## Advanced Examples

### Example 1: Genre-Based Playlists

```bash
# Create genre folders
mkdir -p ~/playlists/{Rock,Jazz,Classical,Electronic}

# Symlink by genre (manually or with script)
ln -s "/media/music/Rock Artist/song.mp3" ~/playlists/Rock/
ln -s "/media/music/Jazz Artist/song.mp3" ~/playlists/Jazz/
```

### Example 2: Rating-Based Smart Playlists

```bash
#!/bin/bash
# Create playlist from 5-star rated tracks

PLAYLIST_DIR=~/playlists/5-Stars
mkdir -p "$PLAYLIST_DIR"

# Find tracks with rating metadata (example using eyeD3)
find /media/music -name "*.mp3" | while read file; do
  rating=$(eyeD3 --no-color "$file" 2>/dev/null | grep -oP 'rating: \K\d+')
  if [ "$rating" = "5" ]; then
    ln -sf "$file" "$PLAYLIST_DIR/"
  fi
done
```

### Example 3: Recently Added Tracks

```bash
# Create playlist from files added in last 7 days
mkdir -p ~/playlists/New-Music
find /media/music -name "*.mp3" -mtime -7 -exec ln -sf {} ~/playlists/New-Music/ \;
```

### Example 4: Mixed Real Files + Symlinks

You can mix both in the same folder:
```bash
mkdir ~/playlists/Mixed
cp /media/music/exclusive.mp3 ~/playlists/Mixed/  # Real file
ln -s /media/music/shared.mp3 ~/playlists/Mixed/  # Symlink
```
Both will be included in the playlist!

---

## Supported Formats

Audio files automatically detected:
- `.mp3`
- `.flac`
- `.m4a`
- `.wav`
- `.ogg`
- `.aac`
- `.wma`
- `.ape`
- `.opus`

---

## Platform-Specific Notes

### Windows

**Symlink Support:**
- ✅ **Symbolic links** - Requires admin OR Developer Mode (Windows 10+)
- ✅ **Junction points** - Works without admin (directories only)
- ✅ **Hard links** - Works without admin (files only, same drive)

**Enable Developer Mode (Windows 10/11):**
1. Settings → Update & Security → For developers
2. Enable **Developer Mode**
3. Restart (may be required)
4. Now symlinks work without admin rights!

**PowerShell Symlink Commands:**
```powershell
# Symbolic link (files)
New-Item -ItemType SymbolicLink -Path "playlist\track.mp3" -Target "D:\music\track.mp3"

# Junction point (directories)
New-Item -ItemType Junction -Path "playlist\folder" -Target "D:\music\folder"

# Hard link (files, same drive, no admin needed)
New-Item -ItemType HardLink -Path "playlist\track.mp3" -Target "D:\music\track.mp3"
```

**Recommendation for Windows Users:**
- Use **hard links** for simplicity (no admin needed)
- Files must be on same drive
- Works with `fs.realpathSync()` just like symlinks

### macOS

**Symlink Support:**
- ✅ **Unix symlinks** (`ln -s`) - Fully supported
- ✅ **Finder aliases** - Automatically resolved by Node.js
- No special permissions needed

### Linux

**Symlink Support:**
- ✅ **Symbolic links** (`ln -s`) - Fully supported
- No special permissions needed

---

## Troubleshooting

### Issue: "No tracks found"

**Check if symlinks are valid:**
```bash
cd ~/playlists/your-folder
ls -la  # Should show: track.mp3 -> /real/path.mp3

# Test individual symlink
test -e track.mp3 && echo "✅ Valid" || echo "❌ Broken"
```

**Check where symlinks point:**
```bash
readlink track.mp3
# Should match a path in your Plex library
```

**Verify Plex has scanned the target files:**
- Go to Plex Web UI
- Search for the track by name
- Check the file path in track details

### Issue: "Only some tracks matched"

**Possible causes:**
- Broken symlinks (target file doesn't exist)
- Wrong file extension (not in supported list)
- Plex hasn't scanned those files yet

**Debug:**
```bash
# Find broken symlinks
find ~/playlists/your-folder -xtype l

# Check all symlink targets
for link in ~/playlists/your-folder/*; do
  echo "$link -> $(readlink "$link")"
  test -e "$link" || echo "  ❌ BROKEN"
done
```

### Issue: "Permission denied"

**macOS:** Grant file access:
1. System Preferences → Security & Privacy
2. Files and Folders → Plex Folder Playlist Creator
3. Enable access to both:
   - Playlist folder (`~/playlists`)
   - Music library (`/media/music`)

**Linux:** Check permissions:
```bash
ls -la ~/playlists/your-folder
# Should be readable by your user
```

### Issue: "Encoding/special character problems"

The app handles common encoding issues automatically:
- Percent-encoded paths (`%20` for space)
- Ampersands (`&` vs `%26`)
- Unicode characters

If still having issues, enable debug logging and check console for warnings.

---

## Tips & Best Practices

### ✅ Do:
- Use absolute paths for symlinks: `ln -s /full/path/file.mp3 dest/`
- Keep symlink names simple (no special characters)
- Organize playlists in dedicated folder: `~/playlists/`
- Use relative symlinks for portable setups

### ❌ Don't:
- Don't move music files after creating symlinks (breaks them)
- Don't use Windows shortcuts (not Unix symlinks)
- Don't nest playlists too deep (keep flat for performance)

### 💡 Pro Tips:

**1. Batch create symlinks from M3U:**
```bash
cat playlist.m3u | while read file; do
  ln -sf "$file" ~/playlists/from-m3u/
done
```

**2. Update symlinks automatically with cron:**
```bash
# crontab -e
0 0 * * * /path/to/update-playlists.sh
```

**3. Use relative symlinks for portable setups:**
```bash
ln -s ../../music/track.mp3 playlists/favorites/
```

**4. Sync playlists across machines:**
```bash
rsync -avL ~/playlists/ remote:~/playlists/
# -L follows symlinks and copies actual files
```

---

## Technical Details

### Detection Logic

1. User selects folder
2. App tries **standard path matching** first (backward compatible)
3. If no matches → automatically tries **symlink resolution**
4. No configuration needed!

### Performance

- **Small playlists (<100 tracks):** Instant
- **Large playlists (1000+ tracks):** ~1-2 seconds
- Only scans when standard matching fails
- Caches nothing (always fresh results)

### Limitations

- **Non-recursive:** Only scans top-level files (not subdirectories)
  - To enable recursion: modify code in `pathUtils.js`
- **Main process only:** Requires Node.js `fs` access (not browser)
- **Same filesystem:** Symlinks must point to accessible paths

---

## Testing

### Quick Test Script

Run the included test script:
```bash
cd ~/git/Plex-Folder-Playlist-Creator
./test-symlinks.sh
```

This creates a test structure in `/tmp` for verification.

### Manual Test

```bash
# 1. Create test structure
mkdir -p /tmp/test-{music,playlist}
echo "test" > /tmp/test-music/track.mp3
ln -s /tmp/test-music/track.mp3 /tmp/test-playlist/

# 2. Launch app and select /tmp/test-playlist
# 3. Check logs for symlink detection
# 4. Cleanup: rm -rf /tmp/test-*
```

---

## Migration from Duplicates

If you currently have duplicate files in playlist folders:

### Step 1: Backup
```bash
cp -r ~/playlists ~/playlists.backup
```

### Step 2: Replace with Symlinks
```bash
#!/bin/bash
for playlist in ~/playlists/*; do
  for file in "$playlist"/*; do
    # Find original in music library
    filename=$(basename "$file")
    original=$(find /media/music -name "$filename" -print -quit)
    
    if [ -n "$original" ]; then
      rm "$file"  # Remove duplicate
      ln -s "$original" "$file"  # Create symlink
    fi
  done
done
```

### Step 3: Recreate Playlists
Re-run playlist creation in the app. Symlinks will now be resolved automatically.

---

## FAQ

**Q: Do I need to configure anything?**  
A: No! Just use symlinks, the app detects and handles them automatically.

**Q: Can I mix symlinks and real files?**  
A: Yes! Both work in the same folder.

**Q: What about macOS aliases?**  
A: Unix symlinks only. Create with `ln -s`, not Finder "Make Alias".

**Q: Can I use relative symlinks?**  
A: Yes, they're resolved to absolute paths automatically.

**Q: What if a symlink is broken?**  
A: It's skipped with a warning in the logs. Other tracks still work.

**Q: Does this work with M3U playlists?**  
A: M3U playlists have their own logic. This feature is for folder-based playlists.

**Q: Can I nest playlist folders?**  
A: Currently non-recursive (top-level only). Can be enabled in code.

**Q: Will this break my existing playlists?**  
A: No! Backward compatible. Standard matching tried first.

---

## See Also

- [Creating Playlists](../README.md#creating-playlists)
- [Bulk Playlist Creation](../README.md#bulk-operations)
- [Troubleshooting Guide](../README.md#troubleshooting)

---

**Happy playlist creating with symlinks! 🎵**
