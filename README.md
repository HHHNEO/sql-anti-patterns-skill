# SQL Anti-Patterns Skill for Claude Code

A comprehensive skill for writing maintainable, performant SQL queries by identifying and avoiding common anti-patterns.

## Overview

This skill provides guidance for Claude Code to help you write clean, maintainable, and performant SQL queries by identifying and avoiding common anti-patterns that lead to:

- Performance degradation
- Maintenance difficulties
- Data integrity issues

Most SQL anti-patterns emerge from time pressure and short-term workarounds that accumulate technical debt. This skill helps identify these patterns early and provides better alternatives.

## What's Included

### 6 Common Anti-Patterns

1. **CASE WHEN Abuse** - Centralize business logic in dimension tables or shared views
2. **Functions on Indexed Columns** - Keep queries sargable for optimal index usage
3. **SELECT * in Views** - Explicit column listing for maintainability
4. **DISTINCT Overuse** - Fix join relationships instead of masking duplicates
5. **Excessive View Nesting** - Flatten hierarchies and materialize complex transformations
6. **Deep Subquery Nesting** - Use CTEs for readable, maintainable queries

### Additional Features

- SQL formatting standards and best practices
- Performance checklist for query deployment
- NULL handling, NOT IN operators, and expression indexes guidance
- Real-world examples with before/after code

## Installation

### For Claude Code

1. Download the skill:
   ```bash
   # Clone this repository
   git clone https://github.com/YOUR_USERNAME/sql-anti-patterns-skill.git

   # Or download the SKILL.md file directly
   ```

2. Copy to your project's skills directory:
   ```bash
   cp -r sql-anti-patterns-skill/.claude/skills/sql-anti-patterns YOUR_PROJECT/.claude/skills/
   ```

3. Or install globally in your user skills directory:
   ```bash
   cp -r sql-anti-patterns-skill/.claude/skills/sql-anti-patterns ~/.claude/skills/
   ```

4. Restart Claude Code or start a new session

### Manual Installation

Simply place the `SKILL.md` file in your project's `.claude/skills/sql-anti-patterns/` directory.

## When to Use

This skill is automatically triggered when:

- Writing new SQL queries, views, or stored procedures
- Reviewing SQL code in pull requests
- Debugging slow query performance
- Refactoring legacy SQL code
- Designing database schemas and views
- Creating database migration scripts
- Optimizing data pipelines
- Resolving data integrity issues from incorrect joins

## Example Usage

When working with Claude Code, simply ask questions or provide code related to SQL:

```
"Review this SQL query for performance issues"
"Help me optimize this slow query"
"Create a view for user order statistics"
"Why is this query doing a full table scan?"
```

Claude will automatically apply the anti-pattern guidelines to provide optimized solutions.

## Key Principles

- **Sargable Queries**: Keep indexed columns "naked" in WHERE clauses
- **Centralized Logic**: Use dimension tables for business logic translations
- **Explicit Definitions**: Always list columns explicitly, especially in views
- **Fix Root Causes**: Address join issues instead of using DISTINCT as a band-aid
- **Flat Structures**: Minimize view nesting and use CTEs for complex queries
- **Production Quality**: Treat SQL as production code requiring design, testing, and maintenance

## Performance Checklist

Before deploying SQL changes:

- [ ] All WHERE clause columns are indexed or sargable
- [ ] No functions applied to indexed columns in predicates
- [ ] JOIN conditions are explicit and use indexed columns
- [ ] DISTINCT usage is justified (not masking join issues)
- [ ] View nesting depth is reasonable (<3 layers)
- [ ] CTEs used for complex logic instead of deep subqueries
- [ ] Query plan reviewed for full table scans
- [ ] Column list is explicit (no SELECT * in production views)

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## License

MIT License - feel free to use this skill in your projects.

## Credits

Based on SQL best practices and anti-pattern discussions from:
- Data Methods Substack article on SQL anti-patterns
- Bill Karwin's "SQL Antipatterns" book
- Community discussions on performance optimization
- Real-world production SQL experiences

## Related Resources

- [Use The Index, Luke](https://use-the-index-luke.com/) - SQL indexing and tuning guide
- [SQL Style Guide](https://www.sqlstyle.guide/) - SQL formatting standards
- [PostgreSQL Performance](https://www.postgresql.org/docs/current/performance-tips.html)
- [SQL Server Query Performance](https://learn.microsoft.com/en-us/sql/relational-databases/performance/performance-monitoring-and-tuning-tools)

---

Made for [Claude Code](https://claude.com/claude-code) by the community
