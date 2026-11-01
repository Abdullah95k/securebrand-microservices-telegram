# securebrand-microservices-telegram

Public Telegram data collection for SecureBrand brand monitoring.

**Status: planning.** There is no code yet. The research and the proposed plan are in
[docs/plan.md](docs/plan.md) and wait for approval before anything is built or run.

Ground rules (from the plan):

- Public data only, read logged out: no Telegram accounts, no bots, no private chats,
  no invite links, no member lists.
- Every Telegram limit is respected: one fixed IP, `robots.txt`, `429` and `Retry-After`
  are obeyed, and there are no proxies or IP rotation.
- Brand monitoring only: Telegram data is never used for AI or machine learning.
- Credentials and session files are never committed, logged or shared. This repository is
  public: never commit captured responses or real data. Test fixtures are synthetic.
