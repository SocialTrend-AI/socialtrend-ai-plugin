# SocialTrend AI

SocialTrend AI connects Claude to your existing SocialTrend AI account so you can manage your content pipeline in conversation: view and reschedule your content calendar, review and act on content waiting for approval, create social post images, newsletters and AI videos, add Knowledge Base sources, and read Auto Engagement leads and activity.

This page is the documentation for the SocialTrend AI connector and plugin.

## Requirements

- A SocialTrend AI account. Sign in with your SocialTrend credentials when you connect.
- Content generation uses the credits and limits of your existing SocialTrend plan. Plans and credits are managed in the SocialTrend app.
- Social accounts you want to post to must already be connected in the SocialTrend app.

## What this plugin contains

- One remote connector: `https://mcp.socialtrend.ai/mcp` (OAuth sign-in).
- Seven skills (approval queue, auto engagement, content calendar, knowledge base, newsletter, social post, video) that guide the assistant on which SocialTrend tools to use and in what order.

## Install

**Claude Code:**

    /plugin install socialtrend-ai --marketplace SocialTrend-AI/socialtrend-ai-plugin

Older Claude Code versions (before 2.1.275):

    /plugin marketplace add SocialTrend-AI/socialtrend-ai-plugin
    /plugin install socialtrend-ai@socialtrend-ai-plugins

**claude.ai / Claude Desktop / Cowork:** Customize → Plugins → Add marketplace → `SocialTrend-AI/socialtrend-ai-plugin`, install **SocialTrend AI**, then open the plugin's **Connectors** tab and sign in with your SocialTrend account.

**Connector only (Claude):** Customize → Connectors → **Add custom connector**, enter `https://mcp.socialtrend.ai/mcp`, then sign in with your SocialTrend account.

## What you can do

| Area | What you can do | Example prompt |
| --- | --- | --- |
| Content calendar | See what is scheduled across social, video, newsletter and blog content, check your posting mix and coverage gaps, and move a scheduled item to a new date or time. | "What's on my content calendar for the next 7 days?" |
| Approval queue | Review content waiting for your approval, then approve it, reject it, or ask for an AI edit. | "Show me everything waiting for my approval." |
| Social post images | Describe the post you want. SocialTrend generates the image and caption, shows a preview, and lets you approve, edit or reject it. | "Create a social post image announcing our weekend open house." |
| Newsletters | Generate a newsletter draft, preview it, and mark it ready in SocialTrend. Sending stays in the SocialTrend app. | "Draft a newsletter about this month's local market update." |
| AI videos | Create a video from one of your saved recipes or a one-off brief, search voices and avatars, check the credit cost before you start, preview the result, and post it to your connected social accounts. | "Make a 30-second video from my listing recipe and tell me the credit cost first." |
| Knowledge Base | Add files, pasted text or web links as sources for your content, and check their scan status. | "Add this to my knowledge base: our office is open Monday to Saturday, 9am to 6pm." |
| Auto Engagement | Read your leads, conversation history, activity feed and engagement stats. | "Show my newest engagement leads from this week." |
| Connected accounts | See which social and video accounts are connected, so you can choose where to post. | "Which social accounts do I have connected?" |

Previews for posts, newsletters and videos render inline where the assistant supports them. Otherwise the assistant gives you a link.

## How it works

- Every action runs on your own SocialTrend account only. The assistant can't see or change anyone else's account.
- Nothing is published without your say-so. Posts and videos go out only to the social accounts you have connected in SocialTrend, and only when you approve or post them. The assistant asks you to confirm before publishing.
- A published post can't be undone from SocialTrend. Remove it on the social platform if needed.

## Good to know

- **Newsletters are sent from the SocialTrend app.** The assistant can draft, edit, reject and mark a newsletter ready, but it can't send it.
- **Video credits.** The assistant checks the credit cost before creating a video. Creating a video needs enough credits and an active video subscription. Approving a video edit re-renders the video and use credits.
- **Video recipes and clone avatars** are created in the SocialTrend app. They can't be created from the assistant.
- **Knowledge Base files** are limited to about 4 MB. Upload larger files in the SocialTrend app. Files and text are scanned for personal information before use.
- **Knowledge Base links** are read at the next Knowledge Base build, so a new link can show as pending for a while. Social media links aren't supported as sources. Rebuilding the Knowledge Base is done in the app.
- **Auto Engagement is read-only.** The assistant can't send messages, reply to comments or change a lead's status. Lead contact details are shown only in the app.
- **Rescheduling** can't move an item into the past. If a move goes over your daily posting cadence, the assistant asks you to confirm first.
- **Times** are shown in your SocialTrend account's timezone (UTC if none is set).
- **Credits and plans** can't be bought or changed from the assistant.

## Troubleshooting

| Problem | What to do |
| --- | --- |
| Sign-in fails or loops | Sign in with the same email and password you use for the SocialTrend app. Then disconnect the connector and connect it again. |
| "Account not connected" or a platform is missing | Reconnect that social account in the SocialTrend app, then ask again. |
| Video creation is blocked | Check your credit balance and video subscription in the SocialTrend app. |
| File too large for the Knowledge Base | Upload it in the SocialTrend app instead. |
| A Knowledge Base link stays pending | Links are read at the next Knowledge Base build; check again later. |
| Preview doesn't show | Open the link the assistant gives you. |
| Newsletter wasn't sent | Send it from the SocialTrend app; the assistant only marks it ready. |
| Anything else | Contact support (below). |

To disconnect, remove the SocialTrend AI connector (or uninstall the plugin) in your assistant's settings.

## Data

The connector sends your requests and the content you provide (for example post briefs, newsletter topics, Knowledge Base text, files or links) to SocialTrend AI's servers to perform the action you ask for. Posts are published only to the social accounts you have connected in SocialTrend, and only when you approve or post them. The plugin contains no scripts or executables and sends nothing anywhere else.

## Links

- Website: [https://socialtrend.ai/](https://socialtrend.ai/)
- Privacy policy: [https://socialtrend.ai/privacy-policy/](https://socialtrend.ai/privacy-policy/)
- Terms of use: [https://socialtrend.ai/terms-of-use/](https://socialtrend.ai/terms-of-use/)
- Support: [https://socialtrend.ai/contact-us/](https://socialtrend.ai/contact-us/)

## License

MIT. See [LICENSE](LICENSE).
