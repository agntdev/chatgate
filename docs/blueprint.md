# Telegram Group Guardian — Bot specification

**Archetype:** community

**Voice:** professional and concise — write every user-facing message, button label, error, and empty state in this voice.

A moderation bot for Telegram groups that automates spam detection and user verification. It enforces rules through configurable thresholds, provides admin commands for manual moderation, and maintains logs and reports of all actions. The bot handles new members with a one-tap verification process, protects against spam through link age, duplicate messages, and flood detection, and ensures admins have full control over settings and exceptions.

> This is the complete contract for the bot. Implement EVERY entry point, flow, feature, integration, and edge case below. The completeness review checks the bot against this document after each build pass.

## Primary audience

- Group owners and admins
- Regular group members

## Success criteria

- New members are verified with one-tap confirmation
- Spam is automatically detected and handled based on configured thresholds
- Admins can manage users and settings through in-group commands
- Action logs and reports are available for audit and review

## Entry points

Every feature must be reachable from the bot's command/button surface (button-first; only /start and /help are slash commands).

- **/start** (command, actor: user, command: /start) — Open the main menu
- **Я человек** (button, actor: user, callback: verify:confirm) — Confirm human verification for new members
  - inputs: Telegram user ID
  - outputs: Verification status updated
- **/warn** (command, actor: admin, command: /warn) — Warn a user with an optional reason
  - inputs: @user, optional reason
  - outputs: Public warning message
- **/mute** (command, actor: admin, command: /mute) — Mute a user for a specified duration with an optional reason
  - inputs: @user, duration, optional reason
  - outputs: Mute restriction applied
- **/kick** (command, actor: admin, command: /kick) — Kick a user with an optional reason
  - inputs: @user, optional reason
  - outputs: User removed from group
- **/ban** (command, actor: admin, command: /ban) — Ban a user with an optional reason
  - inputs: @user, optional reason
  - outputs: User banned from group
- **/trust** (command, actor: admin, command: /trust) — Mark a user as trusted (excluded from checks)
  - inputs: @user
  - outputs: User marked as trusted
- **/untrust** (command, actor: admin, command: /untrust) — Remove a user from trusted list
  - inputs: @user
  - outputs: User removed from trusted list
- **/setgreeting** (command, actor: admin, command: /setgreeting) — Interactive flow to set greeting and button text
  - inputs: Greeting text, Button text
  - outputs: Greeting and button text updated
- **/setrules** (command, actor: admin, command: /setrules) — Interactive flow to set rules text
  - inputs: Rules text
  - outputs: Rules text updated
- **/setthresholds** (command, actor: admin, command: /setthresholds) — Guided sequence to set spam thresholds and select actions
  - inputs: Spam thresholds and actions
  - outputs: Thresholds and actions updated
- **/log** (command, actor: admin, command: /log) — Show recent action log summary in-group or DM to owner
  - inputs: Period
  - outputs: Action log summary
- **/report** (command, actor: admin, command: /report) — Produce overview of joins, verification passes/fails, checks triggered and actions taken
  - inputs: Period
  - outputs: Report summary

## Flows

### New-member verification
_Trigger:_ User joins group

1. Send greeting with verification button
2. Restrict new member until verification or timeout
3. If verified: remove restrictions and log success
4. If timeout: kick user and log removal

_Data touched:_ Member, Pending check, Action log entry

### Spam detection
_Trigger:_ User sends message

1. Check for link from new account
2. Check for repeated messages
3. Check for message flood
4. If threshold exceeded: apply configured action and log

_Data touched:_ Member, Action log entry

### Admin command execution
_Trigger:_ /warn, /mute, /kick, /ban, /trust, /untrust, /setgreeting, /setrules, /setthresholds, /log, /report

1. Verify admin privileges
2. Execute command action
3. Log action if applicable
4. Provide confirmation or error message

_Data touched:_ Member, Rule set, Action log entry, Report

## Data entities

Durable data (must survive a restart) uses the toolkit's persistent store, never in-memory maps.

- **Member** _(retention: persistent)_ — Telegram user in the group with role (admin, trusted, normal, pending verification)
  - fields: Telegram user ID, Role, Verification status, Timestamp
- **Pending check** _(retention: session)_ — Newcomer waiting to confirm via one-tap button with timestamp and timeout
  - fields: Telegram user ID, Timestamp, Timeout
- **Rule set** _(retention: persistent)_ — Greeting text, rules text, verification timeout, spam thresholds, and enabled automatic actions
  - fields: Greeting text, Rules text, Verification timeout, Spam thresholds, Enabled actions
- **Action log entry** _(retention: persistent)_ — Event (warn/mute/kick/ban/delete/remove), target user, reason, trigger (automatic/manual), moderator who issued it, timestamp
  - fields: Event type, Target user, Reason, Trigger, Moderator, Timestamp
- **Report** _(retention: persistent)_ — Aggregated events over a time window
  - fields: Time window, Joins, Verification passes, Verification fails, Checks triggered, Actions taken

## Integrations

- **Telegram** (required) — Bot API messaging and group management
Call external APIs against their real contract (correct endpoints, ids, params); credentials from env. Do not fake responses.

## Owner controls

- Edit greeting and rules text
- Configure automatic actions and thresholds
- Mark users as trusted
- Generate reports of joins, checks, and removals
- Set report delivery preference (private or in-group)

## Notifications

- Greeting message with verification button
- Verification confirmation message
- Spam detection explanation
- Admin command confirmation
- Action log summary
- Report summary

## Permissions & privacy

- Bot requires access to group messages to detect spam
- Bot requires access to user join/leave events
- Bot requires permission to restrict users
- Bot requires permission to send messages in group
- Bot requires permission to delete messages
- Bot requires permission to kick/ban users
- Bot requires permission to pin messages (for rules if needed)

## Edge cases

- User joins and leaves immediately
- User sends multiple messages before verification
- User sends messages with multiple spam signals
- Admin tries to apply action to another admin
- User is marked as trusted but later violates rules
- Bot fails to process message during high load
- User sends message with multiple links from new accounts

## Required tests

- Verify new-member flow with timeout and verification
- Test spam detection with various signals and thresholds
- Validate admin commands with proper permissions and logging
- Generate and review action logs and reports
- Test trusted user exceptions
- Simulate edge cases like rapid joins/leaves and high message volume

## Assumptions

- Verification timeout is set to 3 minutes by default
- Trusted users are implicitly admins and excluded from checks
- Spam thresholds are set to conservative defaults
- Action logs are retained for 90 days
- Reports are delivered privately by default
- Bot will not act on admins or pinned messages
