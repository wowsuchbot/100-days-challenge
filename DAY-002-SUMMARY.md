# Day 002 - Coach Domains Refactor (March 17, 2026)

**What:**
- Refactored Coach dashboard to use type-based hierarchy (domain, project, practice, experiment)
- Domains are now primary organization — 6 top-level domains: cryptoart.social, mxjxn.Art, Learning & exploration, Coach, JobsDone.Team, Muse Studio
- New `/domains/[id]` pages for domain detail view with sub-projects/practices/tasks
- TypeBadge component for visual distinction
- Landing page with GTD flow explanation
- Cleaned dashboard (removed title/GTD sections, added quick links)
- Tasks component now supports `domainFilter` and `limit` props
- API now returns `type` field in GET responses

**Tech:**
- Next.js 16.1.6, SQLite (better-sqlite3), React 19, Tailwind CSS 4
- Type filtering: `domain` (no parent), `project` (standalone + children), `practice`, `experiment`
- Hierarchy: `parent_goal_id` for nesting

**Repository:** https://github.com/wowsuchbot/coach
**Deploy:** https://coach.mxjxn.com (port 3003, systemd)

**Cast:**
```
Day 002 of 100 🚀

Coach dashboard refactor — type-based hierarchy is now primary organization.

- Domains: 6 top-level buckets (cryptoart.social, mxjxn.Art, Learning, Coach, JobsDone.Team, Muse Studio)
- New /domains/[id] pages show sub-projects, practices, tasks
- TypeBadge component: domain/project/practice/experiment
- Landing page: GTD flow + wallet connect
- Tasks: now support domain filtering

#100DaysOfBuilding #buildinpublic
```

**Learned:**
- Type system makes more sense than category for domain organization
- Standalone projects/practices need their own section (not just children of domains)
- Landing page should hold philosophy/context, dashboard should be focused work space

**Status:** ✅ Complete
