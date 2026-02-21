# X Bookmark Briefing

![Status](https://img.shields.io/badge/status-active-success.svg)
![Platform](https://img.shields.io/badge/platform-macOS%20%7C%20Windows-lightgrey.svg)
![License](https://img.shields.io/badge/license-MIT-blue.svg)

Turn your X (Twitter) bookmarks into a daily briefing. Automatically.

## Platform Support

This skill works on both macOS and Windows since the Accomplish app supports both platforms. The skill automatically detects your operating system and uses the right file paths for your system.

## What This Does

You bookmark stuff on X all the time. But do you ever go back and read them? Probably not.

This skill fixes that. It grabs your bookmarks, filters out the junk, organizes everything into categories, and sends you a clean report. Then it saves everything to GitHub so you have a permanent record.

Think of it as your personal bookmark assistant that runs whenever you want.

## How It Works

When you run this skill, here's what happens:

**Step 1: Checks Your Setup**

First, it looks for a config file. The location depends on your operating system:
- macOS: `~/.accomplish/x-briefing/config.json`
- Windows: `%USERPROFILE%\.accomplish\x-briefing\config.json`

If it finds one, great. If not, it'll ask you for your email address and keep going. You can run it without a config file, but having one makes future runs smoother.

**Step 2: Opens Your Bookmarks**

It launches Chrome and goes to your X bookmarks page. If you're not logged in, it'll wait for you to log in. Then it starts scrolling through your bookmarks like a real person would - random pauses, mouse movements, the works. This keeps X from thinking you're a bot.

**Step 3: Grabs Your Bookmarks**

By default, it scrapes 25 bookmarks. But here's the smart part - it remembers what it scraped last time. So if you only have 10 new bookmarks since yesterday, it stops at 10. No wasted time.

It also filters out useless replies. You know those "great post!" replies? Gone. It only keeps replies that have actual links or resources.

If you hit 25 bookmarks and want more, it'll ask if you want to continue. You can grab up to 40 total.

**Step 4: Organizes Everything**

Now it takes all your bookmarks and sorts them into categories like Backend, AI, Hiring, System Design, etc. For each bookmark, it writes a short summary and suggests what you should do with it ("Apply here", "Read this article", etc.).

At the top of the report, you get a summary:
- How many bookmarks total
- How many are new since last time
- How many had external links
- How long the scrape took
- Breakdown by category

**Step 5: Saves Everything**

It saves the report with a filename like `X_Briefing_2024-01-16_0830.md`. The timestamp means you can run this multiple times a day without overwriting anything.

Location depends on your OS:
- macOS: `~/Documents/X_Briefings/`
- Windows: `%USERPROFILE%\Documents\X_Briefings\`

**Step 6: Emails You**

It sends the report to your email using Gmail. You need the google-workspace skill installed for this to work.

**Step 7: Backs Up to GitHub**

Finally, it commits everything to a private GitHub repo called `x-briefings`. If you have the GitHub CLI installed, it handles everything automatically. If not, it'll ask you to create a repo manually and give it the URL.

If there are conflicts (like you ran this on two different computers), it creates a separate branch instead of overwriting anything.

## Setup

### What You Need

1. **Accomplish App** - This is what runs the skill
2. **X Account** - Logged into Chrome
3. **Google Workspace Skill** - For sending emails
4. **GitHub CLI** (optional) - Makes GitHub setup automatic

### Config File

Create a config file at the location for your operating system:

**macOS:** `~/.accomplish/x-briefing/config.json`  
**Windows:** `%USERPROFILE%\.accomplish\x-briefing\config.json`

Content:

```json
{
  "GMAIL_TO_EMAIL": "your@email.com",
  "SCRAPE_LIMIT": 25,
  "AUTO_CONTINUE": false,
  "CATEGORIES": ["Backend", "DevOps", "AI", "Hiring", "System Design", "Resources", "Other"],
  "FILTER_REPLIES": true
}
```

**What each field means:**

- `GMAIL_TO_EMAIL` - Where to send the report (required)
- `SCRAPE_LIMIT` - How many bookmarks to grab initially (default: 25)
- `AUTO_CONTINUE` - Skip the "want to continue?" prompt and auto-grab 15 more (default: false)
- `CATEGORIES` - What categories to organize bookmarks into (default: shown above)
- `FILTER_REPLIES` - Remove low-value replies (default: true)

**Don't have a config file?** No problem. The skill will ask for your email and use default values for everything else. But if you want to run this daily without any prompts, create the config file.

## Using It

Open Accomplish and say:

- "run"
- "Run the X Bookmark Briefing"
- "Check my X bookmarks and email me"

That's it. The skill does the rest.

### First Run

The first time you run it, it'll:
- Ask for your email (if no config)
- Scrape all your bookmarks (up to 25 or 40)
- Create the GitHub repo
- Send you the report

### Daily Runs

On future runs, it's smarter:
- Only grabs NEW bookmarks since last time
- Stops early if there's nothing new
- Updates the same GitHub repo
- Warns you if you're running it too often (less than 6 hours)

## What You Get

A markdown file that looks like this:

```markdown
# X Bookmark Briefing - 2026-02-20 at 08:30

## Summary
- Total: 25 bookmarks
- New: 15 since 2024-01-15
- Filtered: 3 low-value replies
- External Links: 8
- Scrape Time: 45s
- Backend: 7 bookmarks
- AI: 5 bookmarks
- Hiring: 3 bookmarks

---

## Backend

**Mahboob Alam** **(@iMahboob_A)** (2026-02-10): Great article about database indexing strategies...
[Link](https://x.com/...)

Action: Read this article and apply to your current project.

...
```

Clean, organized, and ready to read.

## Features

### Smart Duplicate Prevention

It remembers what you scraped last time. So if you run it twice in one day, it won't give you the same bookmarks again.

### Human-Like Scraping

Random scrolling, mouse movements, and delays. X won't flag you as a bot.

### Automatic Filtering

Skips replies like "Nice!" or "Thanks!" but keeps replies with actual content (links, resources, etc.).

### Progress Updates

You'll see messages like "Scraped 10/25 bookmarks..." so you know it's working.

### Error Recovery

If scraping fails, it waits 5 seconds and tries again. If you're not logged in, it tells you.

### Git Conflict Handling

If you run this on multiple computers, it won't overwrite your work. It creates separate branches when there are conflicts.

## Troubleshooting

**"Browser automation is not available"**

Make sure the Accomplish app is running.

**"Unable to scrape bookmarks"**

Check that you're logged into X on Chrome.

**"Last scrape was 2 hours ago"**

You're running it too often. Wait at least 6 hours between runs to avoid rate limits.

**"No bookmarks found"**

Your bookmark list might be empty, or you might not be logged in.

## Tips

- Run this once a day, maybe in the morning
- If you bookmark a lot, enable `AUTO_CONTINUE` in the config
- Customize the categories to match what you bookmark
- Install GitHub CLI (`brew install gh`) for the smoothest experience

## Files Created

**macOS:**
- `~/Documents/X_Briefings/` - All your reports
- `~/.accomplish/x-briefing/config.json` - Your settings
- `~/.accomplish/x-briefing/last_scrape.json` - Tracks what was scraped (auto-generated)

**Windows:**
- `%USERPROFILE%\Documents\X_Briefings\` - All your reports
- `%USERPROFILE%\.accomplish\x-briefing\config.json` - Your settings
- `%USERPROFILE%\.accomplish\x-briefing\last_scrape.json` - Tracks what was scraped (auto-generated)

## Required Skills

This skill needs the `google-workspace` skill to send emails. Make sure you have it installed in Accomplish.

If it's not available, the skill will tell you to add it.

## License

MIT
