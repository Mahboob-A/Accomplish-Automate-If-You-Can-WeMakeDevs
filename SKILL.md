---
name: "x-bookmark-briefing"
description: "Automates the process of creating a daily briefing from your X (Twitter) bookmarks. It scrapes recent bookmarks, filters out low-value replies, categorizes the content, saves a report, and emails it to you. Finally, it commits and pushes the report to a private GitHub repository (supporting both `gh` automation and manual Git remotes)."
---

# X Bookmark Briefing Skill

## Description
This skill automates the process of creating a daily briefing from your X (Twitter) bookmarks. It scrapes recent bookmarks (capped at 25/40), filters out low-value replies, categorizes the content, saves a report, and emails it to you. Finally, it commits and pushes the report to a private GitHub repository (supporting both `gh` automation and manual Git remotes).

## Behavior
User can invoke this skill with simple commands:
- "run"
- "Run the X Bookmark Briefing"
- "Check my X bookmarks and email me a summary"
- "Scrape my bookmarks, summarize them, and push to GitHub"

**Agent Behavior:**
When this skill is invoked, the agent must:
1. Execute the complete workflow (scrape → analyze → save → email → git push)
2. Automatically use the `/google-workspace` skill for email delivery (no need for user to specify)
3. Handle all steps autonomously unless user input is required (e.g., missing config, login needed, or optional extended scraping)

## Prerequisites
- **Required Skill:** `/google-workspace` (for email delivery)
- **Agent Instruction:** If `/google-workspace` is not available when attempting to send email, inform the user: *"This skill requires the `/google-workspace` skill to send emails. Please add it to continue."*


## Workflow Steps

### 1. Configuration & Directory Check
**Agent Instruction:**
1.  **Detect Operating System:**
    -   Determine the current OS (macOS, Windows).
    -   Set base paths accordingly:
        -   **macOS:** `OUTPUT_DIR = ~/Documents/X_Briefings/`, `CONFIG_DIR = ~/.accomplish/x-briefing/`
        -   **Windows:** `OUTPUT_DIR = %USERPROFILE%\Documents\X_Briefings\`, `CONFIG_DIR = %USERPROFILE%\.accomplish\x-briefing\`
2.  **Check Config:** Look for config file at `{CONFIG_DIR}/config.json`.
    -   **Expected format:**
        ```json
        {
          "GMAIL_TO_EMAIL": "your@email.com",
          "SCRAPE_LIMIT": 25,
          "AUTO_CONTINUE": false,
          "CATEGORIES": ["Backend", "DevOps", "Platform", "Agents", "DSA", "AI", "Hiring", "System Design", "Resources", "Other"],
          "FILTER_REPLIES": true
        }
        ```
    -   **If missing:** 
        -   Inform the user: *"For future automation, please create the config file at `{CONFIG_DIR}/config.json` with your email: `{\"GMAIL_TO_EMAIL\": \"your@email.com\"}`"*
        -   Ask: *"What email address should I use for this briefing?"*
        -   Store the provided email for use in Step 5.
        -   If possible, create the config file with provided email and default values.
    -   **If exists:** 
        -   Read `GMAIL_TO_EMAIL` (required)
        -   Read optional: `SCRAPE_LIMIT` (default: 25), `AUTO_CONTINUE` (default: false), `CATEGORIES` (default list), `FILTER_REPLIES` (default: true)
        -   Store all values for use in subsequent steps.
3.  **Check Output Directory:**
    -   Target: `{OUTPUT_DIR}` (OS-specific path from Step 1.1)
    -   Check if it exists. If not, create it.
4.  **Load Checkpoint (Duplicate Prevention):**
    -   Check for `{CONFIG_DIR}/last_scrape.json`.
    -   **If exists:** Read `lastScrapedIds` array, `lastScrapeDate`, and `lastScrapeTimestamp`.
    -   **If missing:** Set `lastScrapedIds = []`, `lastScrapeDate = null`, `lastScrapeTimestamp = null` (first run).
    -   Store these values for use in Step 1.5 and Step 2.

### 1.5. Pre-flight Validation
**Agent Instruction:**
1.  **Verify Browser Automation:**
    -   Check if browser automation is available (Accomplish app running).
    -   If not available: Inform user *"Browser automation is not available. Please ensure the Accomplish app is running."* and stop.
2.  **Validate Email Format:**
    -   Check if `GMAIL_TO_EMAIL` is a valid email format (contains @ and domain).
    -   If invalid: Ask user to provide a valid email address.
3.  **Check Scrape Frequency:**
    -   If `lastScrapeTimestamp` exists, calculate time difference from now.
    -   If less than 6 hours: Warn user *"Last scrape was {X} hours ago. Running too frequently may trigger rate limits. Continue anyway?"*
    -   If user says NO: Stop workflow.
    -   If user says YES or time > 6 hours: Continue.

### 2. Browser Scraping (Strict Limit & Human Behavior)
**Agent Instruction:**
1.  Launch the browser and navigate to `https://x.com/i/bookmarks`.
2.  **Check for login:** If missing, ask user to log in and wait.
3.  **Phase A: Scrape Initial Batch**
    -   Use `SCRAPE_LIMIT` from config (default: 25).
    -   Execute the script below with `LIMIT = SCRAPE_LIMIT` and `lastScrapedIds` from Step 1.
    -   If `lastScrapedIds` is not empty, inform user: *"Checking for new bookmarks since {lastScrapeDate}..."*
    -   **Show progress:** Update user every 5 bookmarks: *"Scraped {current}/{SCRAPE_LIMIT} bookmarks..."*
4.  **Handle Scraping Results:**
    -   **If scraping fails (0 bookmarks and errors exist):**
        -   Inform user: *"Scraping failed. Retrying in 5 seconds..."*
        -   Wait 5 seconds and retry once.
        -   If still fails: *"Unable to scrape bookmarks. Please verify you're logged in and try again."* Stop workflow.
    -   **If 0 bookmarks scraped (no errors):**
        -   Inform user: *"No bookmarks found. Your bookmark list may be empty."* Stop workflow.
5.  **Phase B: Smart Continuation Logic**
    -   **If `hitCheckpoint = true`:** 
        -   Inform user: *"Found {count} new bookmarks since last scrape. Filtered out {filteredCount} low-value replies."*
        -   Proceed to Step 3.
    -   **If scraped SCRAPE_LIMIT without hitting checkpoint:**
        -   If `AUTO_CONTINUE = true`: Automatically continue with 15 more.
        -   If `AUTO_CONTINUE = false`: Ask user: *"I have scraped {SCRAPE_LIMIT} bookmarks. Do you want me to continue scraping more (max 15 more)?"*
        -   **If YES:** Execute script with `LIMIT = 15`, `MODE = "extra_slow"`, and `lastScrapedIds`. Combine results. Show progress.
        -   **If NO:** Proceed with the scraped bookmarks.

**Scraper Script Logic:**
```javascript
async function scrapeBookmarks(limit = 25, mode = "normal", lastScrapedIds = [], filterReplies = true) {
  const tweets = [];
  const seenIds = new Set();
  const errors = [];
  const filtered = { replies: 0, total: 0 };
  let hitCheckpoint = false;
  let checkpointId = null;
  
  const wait = (min, max) => new Promise(r => setTimeout(r, Math.floor(Math.random() * (max - min + 1)) + min));
  
  const simulateHumanInteraction = async () => {
    const scrollAmount = Math.floor(Math.random() * 100) - 50;
    window.scrollBy(0, scrollAmount);
    
    if (Math.random() > 0.7) await wait(500, 1500);
    
    const elements = document.querySelectorAll('article');
    if (elements.length > 0) {
      const randomEl = elements[Math.floor(Math.random() * elements.length)];
      randomEl.dispatchEvent(new MouseEvent('mouseover', { bubbles: true }));
    }
  };

  const extractLinks = (article) => {
    const links = [];
    const linkElements = article.querySelectorAll('a[href^="http"]:not([href*="/status/"])');
    linkElements.forEach(el => {
      const href = el.href;
      if (!href.includes('t.co') && !href.includes('twitter.com') && !href.includes('x.com')) {
        links.push(href);
      }
    });
    return links;
  };

  let noNewTweetsCount = 0;
  let scrollAttempts = 0;
  const maxScrollAttempts = mode === "extra_slow" ? 20 : 15;
  
  while (tweets.length < limit && noNewTweetsCount < 3 && scrollAttempts < maxScrollAttempts && !hitCheckpoint) {
    const articles = document.querySelectorAll('article[data-testid="tweet"]');
    let addedNew = false;

    for (const article of articles) {
      if (tweets.length >= limit || hitCheckpoint) break;
      
      try {
        const textEl = article.querySelector('[data-testid="tweetText"]');
        if (!textEl) continue;

        const text = textEl.innerText;
        const userEl = article.querySelector('[data-testid="User-Name"]');
        const linkEl = article.querySelector('a[href*="/status/"]');
        const timeEl = article.querySelector('time');
        const link = linkEl ? linkEl.href : null;
        
        // Check if we hit a previously scraped tweet
        if (link && lastScrapedIds.length > 0 && lastScrapedIds.includes(link)) {
          hitCheckpoint = true;
          checkpointId = link;
          break;
        }
        
        const isReply = text.includes("Replying to @"); 
        let shouldKeep = true;

        if (filterReplies && isReply) {
          const keywords = ["http", "github", "resource", "hiring", "job", "guide", "learn", "tool", "ai", "platform", "check this", "read", "article", "blog", "tutorial", "course"];
          const hasKeyword = keywords.some(k => text.toLowerCase().includes(k));
          if (!hasKeyword) {
            shouldKeep = false;
            filtered.replies++;
          }
        }
        
        filtered.total++;

        if (shouldKeep && link && !seenIds.has(link)) {
           seenIds.add(link);
           const externalLinks = extractLinks(article);
           
           tweets.push({
             text: text,
             author: userEl ? userEl.innerText.split('\n')[0] : "Unknown",
             handle: userEl ? userEl.innerText.split('\n')[1] : "Unknown",
             time: timeEl ? timeEl.getAttribute('datetime') : new Date().toISOString(),
             link: link,
             externalLinks: externalLinks,
             isReply: isReply
           });
           addedNew = true;
        }
      } catch (e) { 
        errors.push({ error: e.message, timestamp: new Date().toISOString() });
      }
    }

    if (hitCheckpoint) break;

    if (!addedNew) noNewTweetsCount++;
    else noNewTweetsCount = 0;

    await simulateHumanInteraction();
    
    const scrollFactor = mode === "extra_slow" ? 0.4 : 0.7; 
    window.scrollBy(0, window.innerHeight * (scrollFactor + Math.random() * 0.2));
    
    const waitTime = mode === "extra_slow" ? [3000, 6000] : [2000, 4000];
    await wait(waitTime[0], waitTime[1]);
    
    scrollAttempts++;
  }
  
  return { 
    count: tweets.length, 
    tweets: tweets,
    errors: errors.length > 0 ? errors : undefined,
    reachedLimit: tweets.length >= limit,
    hitCheckpoint: hitCheckpoint,
    checkpointId: checkpointId,
    filtered: filtered
  };
}
// agent will inject: return await scrapeBookmarks(SCRAPE_LIMIT, "normal", lastScrapedIds, FILTER_REPLIES);
return await scrapeBookmarks(25, "normal", [], true); 
```

### 3. Analysis & Summarization
**Agent Instruction:**
1.  **Calculate Statistics:**
    -   Total bookmarks: {count}
    -   New bookmarks: {count if lastScrapeDate exists, else same as total}
    -   External links: Count bookmarks with externalLinks array length > 0
    -   Scrape duration: Calculate from start to end of scraping
    -   **Dynamic category counts:** For each category in CATEGORIES, count how many bookmarks will be assigned to it based on content analysis

2.  **Pass the combined JSON output to the LLM with context:**

> "Analyze these X bookmarks. 
> {If lastScrapeDate exists: 'These are {count} new bookmarks since {lastScrapeDate}.'}
> Group them into categories from config: {CATEGORIES}.
> 
> IMPORTANT: Start the report with this exact structure (NO EMOJIS):
> 
> # X Bookmark Briefing - {YYYY-MM-DD} at {HH:MM}
> 
> ## Summary
> - Total: {count} bookmarks
> {If lastScrapeDate: '- New: {count} since {lastScrapeDate}'}
> {If filtered.replies > 0: '- Filtered: {filtered.replies} low-value replies'}
> - External Links: {count with externalLinks}
> - Scrape Time: {duration}s
> {For each category with bookmarks: '- {CategoryName}: {count} bookmarks'}
> 
> ---
> 
> Then for each category with bookmarks:
> ## {Category Name}
> 
> For each bookmark:
> - Format: `**Author** (Date): Summary (30-50 words). [Link](url)`
> - **Action Items:** Explicitly suggest next steps (e.g., 'Apply here', 'Clone this repo').
> 
> Output a clean and properly formatted Markdown report with NO EMOJIS."

### 4. File Generation
**Agent Instruction:**
1.  **Generate unique filename with timestamp:**
    -   Format: `X_Briefing_YYYY-MM-DD_HHMM.md` (e.g., `X_Briefing_2026-02-20_0830.md`)
    -   Use 24-hour format for time (HH:MM)
    -   This prevents overwriting if multiple runs happen on the same day.
2.  **Save File:** `{OUTPUT_DIR}/X_Briefing_YYYY-MM-DD_HHMM.md` (using OS-specific path from Step 1.1)
3.  **Store the timestamp** for use in Step 6 (git commit and branch naming).

### 4.5. Save Checkpoint (Duplicate Prevention)
**Agent Instruction:**
1.  Extract the first 5 tweet links from the scraped results. If fewer than 5 bookmarks were scraped, use as many as available.
2.  Create/update `{CONFIG_DIR}/last_scrape.json` (using OS-specific path from Step 1.1):
    ```json
    {
      "lastScrapedIds": ["link1", "link2", "link3", "link4", "link5"],
      "lastScrapeDate": "YYYY-MM-DD",
      "lastScrapeTimestamp": "YYYY-MM-DDTHH:MM:SSZ",
      "totalScraped": <count>
    }
    ```
3.  Ensure the directory `{CONFIG_DIR}` exists before writing.

### 5. Email Delivery
**Agent Instruction:**
1.  Use the email obtained from Step 1 (either from config file or user input).
2.  Send the email via `/google-workspace` skill.

### 6. Git & GitHub Automation
**Agent Instruction:**
1.  **Directory:** `{OUTPUT_DIR}` (using OS-specific path from Step 1.1)
2.  **Git Init:** Check if `.git` exists. If not, run `git init`.
3.  **Check Remote:** Run `git remote -v`.
4.  **Remote Setup:**
    -   If remote exists: Continue.
    -   If no remote:
        -   Try `gh auth status`.
        -   **If `gh` works:** `gh repo create x-briefings --private --source=. --remote=origin`.
        -   **If `gh` fails or is missing:**
            -   Notify user: *"I cannot create the GitHub repo automatically because the GitHub CLI (`gh`) is missing or not authenticated. Please create a repository manually on GitHub."*
            -   Ask user: *"What is the new repository URL?"*
            -   Run `git remote add origin [user_provided_url]`.
5.  **Smart Push with Conflict Handling:**
    -   `git add .`
    -   `git commit -m "Briefing for {YYYY-MM-DD} at {HH:MM}"`
    -   **Check for remote changes:** `git fetch origin`
    -   **If local is behind remote (conflicts detected):**
        -   Create timestamped branch: `git checkout -b briefing-{YYYY-MM-DD}-{HHMM}`
        -   Push to branch: `git push origin briefing-{YYYY-MM-DD}-{HHMM}`
        -   Inform user: *"Pushed to branch 'briefing-{YYYY-MM-DD}-{HHMM}' due to remote changes. Please merge manually."*
    -   **If no conflicts:**
        -   Push to main: `git push origin main`
        -   Inform user: *"Successfully pushed briefing to GitHub (main branch)."*
