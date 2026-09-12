# Daily Study Plan

Powers the daily 10:00 AM IST flashcard + quiz delivered through the Claude app.

- `topics.json` -- all 109 study topics from the repo, in the correct learning order (Module 1 -> Module 12, following each module's own README sequence). Each entry has a `day` number, `module`, `title`, and file `path`.
- `progress.json` -- tracks where you are: `topic_cursor` (index into `topics.json` for the next new topic), `session_count` (total daily runs so far), `pending_quiz` (the most recently delivered quiz + answer key, used for grading when you reply), and `history` (a log of every day's topic, delivered time, and quiz score once graded).

## How it works

1. Every day at 10:00 AM IST, a scheduled Claude session reads `progress.json` to see where you left off.
2. It picks the next topic from `topics.json`, reads the real study file, and condenses it into a short, fun "flashcard" (analogy, key points, why it matters for the exam) -- no need to open GitHub, it's delivered straight in the chat.
3. It follows the card with a mini-quiz: 2 multiple-choice questions plus 1 CTF-style scenario question.
4. Every 7th day is a **review day** instead of new material: a mixed quiz across the last week's topics, to reinforce retention.
5. Whenever you reply with your answers (same chat thread, any time), it grades them, explains the correct answer, and logs your score to `history`.
6. `topic_cursor` only advances after a normal study day (not a review day), so you always land on 109 completed topics after roughly 109 study days + ~16 review days (~4 months at this pace).

## Adjusting the plan

Just tell Claude in chat -- e.g. "speed up to 2 topics/day," "skip review days," "pause the daily reminder for a week," "change the time to 8pm." It can update the scheduled task and these files directly.
