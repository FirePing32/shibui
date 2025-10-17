---
title: "Automated Email Spam Detection Using N8N"
date: 2025-10-18T17:12:11+05:30
comment: false
tags: ["n8n", "automation", "agents"]
---

# How I Built an AI-Powered Spam Filter That Actually Works

I was drowning in spam emails. Every morning, I'd wake up to dozens of "You've won a prize!" messages, sketchy invoices, and promotional garbage that somehow slipped past Gmail's filters. I spent way too much time manually cleaning my inbox, and I knew there had to be a better way.

That's when I decided to build my own AI-powered spam cleanup system using N8N. The idea was simple: let an AI agent analyze incoming emails and automatically delete the spam while notifying me on Slack. No more manual cleanup, no more wasted time.

Here's what I ended up building:
- A Gmail trigger that monitors my inbox in real-time
- An AI agent that analyzes emails for spam signals
- Automatic deletion of confirmed spam
- Slack notifications so I know what's being removed
- Smart filtering that leaves legitimate emails untouched

Spoiler alert: it works incredibly well. Let me show you how I built it.

## What You'll Need

Before I get into the technical details, here's what I used:
- N8N (I'm running it on a cloud instance, but self-hosted works too)
- My Gmail account
- A Slack workspace for notifications
- OpenAI API access (though you can use Claude, Gemini, or any AI model)

The whole setup took me about an hour to get working, and another few days of tweaking to make it perfect.

## The Architecture

My workflow is pretty straightforward:

```
Gmail Trigger → AI Agent Analysis → Decision Node → Delete Email + Slack Notification
                                                   → Do Nothing
```

Simple, but effective. Let me walk you through each part.

---

## Setting Up Gmail OAuth (The Tedious But Necessary Part)

The first hurdle I faced was getting N8N to talk to Gmail. This meant diving into Google Cloud Console and setting up OAuth credentials. It's a bit tedious, but I only had to do it once, so here's how I got through it.

### Getting Google Cloud Credentials

First, I needed to create OAuth credentials in Google Cloud Console. Here's what I did:

1. **Created a Google Cloud Project**
   - Headed over to [console.cloud.google.com](https://console.cloud.google.com)
   - I already had a project, but you can create a new one if needed

2. **Enabled the Gmail API**
   - Navigated to "APIs & Services" > "Library"
   - Searched for "Gmail API"
   - Hit "Enable" - this is what allows N8N to interact with Gmail

3. **Set Up OAuth Consent Screen**
   - Went to "APIs & Services" > "OAuth consent screen"
   - Chose "External" user type (unless you're on Google Workspace)
   - Filled in the basics:
     - App name: I used "N8N Gmail Automation"
     - User support email: My email
     - Developer contact: Also my email
   - Added the scope: `https://www.googleapis.com/auth/gmail.modify`
   - This scope is crucial - it lets the workflow both read and delete emails

4. **Generated OAuth 2.0 Credentials**
   - Navigated to "APIs & Services" > "Credentials"
   - Clicked "Create Credentials" > "OAuth 2.0 Client ID"
   - Selected "Web application"
   - Added my N8N redirect URI:
     - For N8N Cloud: `https://app.n8n.cloud/rest/oauth2-credential/callback`
     - For self-hosted: `https://your-n8n-domain.com/rest/oauth2-credential/callback`
   - Saved the Client ID and Client Secret (I'll need these in a minute)

### Connecting N8N to Gmail

With credentials in hand, I jumped into N8N:

1. **Created the Credential**
   - In my N8N workflow editor, I added a "Gmail" node
   - Clicked "Credential to connect with" dropdown
   - Selected "Create New Credential"
   - Chose "Gmail OAuth2 API"

2. **Plugged in the OAuth Details**
   - Pasted my Client ID from Google Cloud
   - Pasted my Client Secret
   - The Auth URI and Token URI were already filled in:
     - Auth URI: `https://accounts.google.com/o/oauth2/v2/auth`
     - Token URI: `https://oauth2.googleapis.com/token`
   - Scope: `https://www.googleapis.com/auth/gmail.modify`

3. **Connected My Account**
   - Clicked "Connect my account"
   - A popup opened asking me to authorize
   - Selected my Google account
   - Reviewed the permissions and accepted
   - Got a success message - we're in business!

### Setting Up the Trigger

Now for the fun part - making N8N listen for new emails:

1. I added a "Gmail Trigger" node to my workflow
2. Configured it like this:
   - **Event**: "Message Received"
   - **Label Names**: Left empty to monitor everything (you could specify "INBOX" or other labels)
   - **Polling Interval**: Set to 5 minutes (adjust based on how real-time you need it)
   - **Simple**: Turned this OFF to get the full email body, not just metadata

3. Tested it by sending myself an email and clicking "Listen for event"

It worked! I could see the email data flowing into N8N. One thing I learned the hard way: for production, Gmail's push notifications are way better than polling. They're real-time and don't eat up your API quota as fast.

---

## Crafting the AI Prompt (The Most Important Part)

Here's where things got interesting. The quality of my spam detection would live or die by the prompt I gave the AI. I spent a good chunk of time iterating on this, and I learned that being specific and structured is key.

### Adding the AI Node

I added an OpenAI node to the workflow (you could use Claude or any other AI model), selected "Message a model" as the operation, and went with GPT-4 for better accuracy. Yes, it costs more, but fewer false positives are worth it to me.

### The Prompt I Settled On

After a bunch of trial and error, here's the prompt that gave me the best results:

```
You are an advanced email spam detection system. Your task is to analyze the provided email and determine if it's spam.

Analyze the following email carefully:

Subject: {{$json.subject}}
From: {{$json.from.email}}
From Name: {{$json.from.name}}
Date: {{$json.date}}
Body: {{$json.textPlain}}

Evaluation Criteria:
1. Sender Reputation: Is the sender from a legitimate domain?
2. Content Analysis: Does the email contain typical spam indicators like excessive capital letters, urgent calls to action, or too-good-to-be-true offers?
3. Links and URLs: Are there suspicious shortened links or domains?
4. Personalization: Is the email generic or personalized?
5. Grammar and Spelling: Poor grammar is often a spam indicator
6. Legitimate Business: Does this appear to be from a real business or service?

Important Guidelines:
- Newsletters and promotional emails from legitimate companies (Amazon, LinkedIn, etc.) should NOT be marked as spam
- Emails from known services or platforms should be preserved
- Be conservative - when in doubt, mark as NOT spam to avoid false positives
- Focus on clear spam signals: phishing attempts, scams, unsolicited offers, fake invoices

You must respond with a valid JSON object only, with no additional text or explanation:

{
  "is_spam": true/false,
  "confidence": 0-100,
  "reason": "Brief explanation of your decision",
  "category": "phishing/scam/promotional/legitimate/suspicious"
}

Respond only with the JSON object.
```

### Why This Works for Me

I spent way too much time tweaking this prompt, but here's why this version works so well:

1. **Clear Context**: The AI knows exactly what I'm asking it to do
2. **Real Email Data**: Using N8N's expression syntax, I inject actual email metadata
3. **Structured Thinking**: I give it a framework to analyze emails, not just a yes/no question
4. **Conservative Guidelines**: This is crucial - I explicitly tell it to err on the side of caution. Better to let some spam through than delete an important email
5. **JSON Output**: By forcing structured output, I can parse it reliably in the next step
6. **Confidence Scoring**: This lets me set thresholds later - I can be more aggressive with 95% confidence emails

The key learning for me was the "Important Guidelines" section. Initially, I had the AI flagging legitimate newsletters and promotional emails from companies I actually do business with. Adding explicit instructions to preserve those made a huge difference.

### Customizing for Your Needs

Since building this, I've tweaked it a few times:

- I added a list of domains I never want flagged (company emails, important services)
- I increased the emphasis on phishing detection after getting some sketchy invoice emails
- I adjusted it to handle emails in multiple languages since I get some non-English spam

You'll probably want to iterate on this based on the spam patterns you see.

---

## Parsing the AI Response (JavaScript to the Rescue)

The AI returns its analysis as a string, but I needed structured data to make decisions. This is where I added a Code node with some JavaScript to parse the response and handle edge cases.

### The Code Node Setup

I added a Code node right after the AI node, set it to "Run Once for All Items", and wrote this JavaScript:

```javascript
// Get the AI response from the previous node
const aiResponse = $input.item.json.message.content;

// Function to extract JSON from response (handles cases where AI adds extra text)
function extractJSON(text) {
  // Try to find JSON object in the text
  const jsonMatch = text.match(/\{[\s\S]*\}/);
  if (jsonMatch) {
    return jsonMatch[0];
  }
  return text;
}

// Function to safely parse JSON with error handling
function safeJSONParse(text) {
  try {
    const cleanedText = extractJSON(text);
    return JSON.parse(cleanedText);
  } catch (error) {
    console.error('JSON Parse Error:', error);
    console.error('Attempted to parse:', text);
    // Return a safe default if parsing fails
    return {
      is_spam: false,
      confidence: 0,
      reason: 'Failed to parse AI response - defaulting to not spam for safety',
      category: 'error',
      parse_error: true
    };
  }
}

// Parse the AI response
const parsedResponse = safeJSONParse(aiResponse);

// Get original email data from the Gmail trigger
const emailData = $('Gmail Trigger').item.json;

// Construct the output object with all relevant information
const output = {
  // Spam detection results
  is_spam: parsedResponse.is_spam,
  confidence: parsedResponse.confidence,
  reason: parsedResponse.reason,
  category: parsedResponse.category,
  
  // Original email metadata for later use
  email_id: emailData.id,
  email_subject: emailData.subject,
  email_from: emailData.from.email,
  email_from_name: emailData.from.name,
  email_date: emailData.date,
  
  // Snippet for Slack notification
  email_snippet: emailData.snippet || emailData.textPlain.substring(0, 150),
  
  // Error tracking
  had_parse_error: parsedResponse.parse_error || false,
  
  // Timestamp for logging
  processed_at: new Date().toISOString()
};

// Return the structured data
return [output];
```

### What This Script Does

Let me break down what's happening here, because I ran into some gotchas:

1. **Gets the AI Response**: Pulls the content from the previous OpenAI node

2. **Extracts JSON**: The `extractJSON()` function was a lifesaver. Sometimes the AI would add extra text like "Here's the analysis:" before the JSON, and this regex finds the actual JSON object

3. **Safe Parsing**: I wrapped everything in try-catch because early on, I had the workflow crash when the AI returned malformed JSON. Now it defaults to "not spam" if anything goes wrong - way safer

4. **Error Handling**: This is critical. If parsing fails, it marks the email as NOT spam. I'd rather manually delete one spam email than accidentally delete something important

5. **Data Enrichment**: I combine the AI analysis with the original email data so I have everything I need in one object for downstream nodes

6. **Accessing Previous Nodes**: That `$('Gmail Trigger').item.json` syntax lets me grab data from earlier in the workflow

### Testing This Out

When I first built this, I tested it by:

1. Running the workflow with a test email
2. Clicking on the Code node after execution
3. Checking the output to make sure all fields were there

I caught a few bugs this way - like trying to access properties that didn't exist on certain emails. Adding fallbacks and optional chaining (`?.`) fixed those issues.

---

## Making Decisions (The IF Node)

Now that I had structured data about whether an email was spam, I needed to route the workflow accordingly. This is where N8N's IF node came in clutch.

### Setting Up the Conditional Logic

I added an IF node after the Code node. This splits the workflow into two paths: one for spam, one for legitimate emails.

Here's how I configured it:

**Condition 1: Check if it's spam**
- **Field**: `{{ $json.is_spam }}`
- **Operation**: "Equal"
- **Value**: `true`

**Condition 2: Confidence threshold** (I added this later after some testing)
- Clicked "Add Condition"
- **Combine**: "AND"
- **Field**: `{{ $json.confidence }}`
- **Operation**: "Larger"
- **Value**: `70`

So the email needs to be flagged as spam AND the AI needs to be at least 70% confident. This prevents false positives.

### Why the Confidence Threshold Matters

I learned this the hard way. Initially, I didn't have a confidence check, and the AI would sometimes mark legitimate emails as spam with like 55% confidence. That's basically a coin flip.

Now I only act on high-confidence decisions:

- **90-100% confident**: Definitely spam, delete it
- **70-89% confident**: Probably spam, safe to delete
- **Below 70%**: Too uncertain, better to keep it

I've been running this for a few months now and 70% seems to be the sweet spot. I haven't had a false positive yet.

### What Happens on Each Path

#### True Path: Delete and Notify

When an email is confirmed spam, I do two things:

**1. Delete it from Gmail**

I added a Gmail node to the true path:
- **Operation**: "Delete"
- **Message ID**: `{{ $json.email_id }}`

This permanently removes the email. No trash, no recovery. Gone.

**2. Notify myself on Slack**

Right after the delete, I added a Slack node:
- **Operation**: "Post Message"
- **Channel**: I use #spam-alerts
- **Message**: Here's what I send:

```
🗑️ Spam Email Deleted

*Subject:* {{ $json.email_subject }}
*From:* {{ $json.email_from_name }} <{{ $json.email_from }}>
*Category:* {{ $json.category }}
*Confidence:* {{ $json.confidence }}%

*Reason:* {{ $json.reason }}

*Preview:* {{ $json.email_snippet }}

*Deleted at:* {{ $json.processed_at }}
```

This gives me full visibility into what's being deleted. I check this channel once a day to make sure nothing important got caught.

I actually went back later and made it fancier using Slack's Block Kit format. It looks way more professional:

```json
{
  "blocks": [
    {
      "type": "header",
      "text": {
        "type": "plain_text",
        "text": "🗑️ Spam Email Deleted"
      }
    },
    {
      "type": "section",
      "fields": [
        {
          "type": "mrkdwn",
          "text": "*Subject:*\n{{ $json.email_subject }}"
        },
        {
          "type": "mrkdwn",
          "text": "*From:*\n{{ $json.email_from }}"
        },
        {
          "type": "mrkdwn",
          "text": "*Confidence:*\n{{ $json.confidence }}%"
        },
        {
          "type": "mrkdwn",
          "text": "*Category:*\n{{ $json.category }}"
        }
      ]
    },
    {
      "type": "section",
      "text": {
        "type": "mrkdwn",
        "text": "*Reason:* {{ $json.reason }}"
      }
    }
  ]
}
```
