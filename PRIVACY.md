# Privacy Policy

gmail-assistant is a personal, self-hosted Gmail triage tool. This policy
describes how it handles data.

## What the app accesses

- With explicit OAuth consent per account, the app accesses Gmail via the
  official Gmail API, using the scopes `gmail.modify` and `gmail.send`.
- It reads message metadata (sender, subject, list headers, labels, snippet).
- Full message bodies are read only when composing a draft or sending, at the
  user's explicit request.

## Where data goes

- Message metadata, OAuth tokens, and audit logs are stored **locally** on the
  owner's own machines (SQLite files and JSON logs). Nothing is uploaded to
  any third-party service by this app.
- Sender-level classification (address, domain, list-id, message count, and a
  sample subject line — never message bodies) is sent exclusively to a
  self-hosted language-model server on the owner's private local network.
- The app sends email only when the user explicitly composes and confirms a
  send. It has no analytics, no telemetry, and no remote endpoints other than
  Google's Gmail API and the owner's own local model server.

## Data retention

- Local files persist until the user deletes them.
- OAuth tokens can be revoked at any time at
  <https://myaccount.google.com/permissions> or with the app's
  `gmailcleaner auth revoke` command.

## Contact

nestor.mata@gmail.com
