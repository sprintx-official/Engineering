# Boilerplate Overview

A comprehensive collection of production-ready boilerplates for modern web development.

## Available Boilerplates

### 1. React.js Boilerplate
**Repository:** [reactjs-boilerplate](https://github.com/sprintx-official/reactjs-boilerplate)

**Technology Stack:**
- React 19 with TypeScript 5
- Vite 6 for build tooling
- ESLint 9 + Prettier 3
- TanStack Router for type-safe routing
- Husky for git hooks

**Use Cases:**
- Single Page Applications (SPA)
- Client-side web applications
- Interactive dashboards and admin panels
- Progressive Web Apps (PWA)

**Key Benefits:**
- Fast development with Vite HMR
- Strict TypeScript for type safety
- Modern React patterns and hooks
- Automated code quality checks

---

### 2. Next.js Boilerplate
**Repository:** [nextjs-boilerplate](https://github.com/sprintx-official/nextjs-boilerplate)

**Technology Stack:**
- Next.js 15 with App Router
- TypeScript 5 (strict mode)
- ESLint 9 + Prettier 3
- Tailwind CSS or styled-components
- Husky for git hooks

**Use Cases:**
- Server-side rendered applications
- Static site generation
- Full-stack applications
- SEO-critical websites
- E-commerce platforms

**Key Benefits:**
- Built-in server-side rendering
- Automatic code splitting
- Optimized production builds
- API routes for backend logic
- Superior SEO performance

---

### 3. Node.js + Express Boilerplate
**Repository:** [node-express-boilerplate](https://github.com/sprintx-official/node-express-boilerplate)

**Technology Stack:**
- Express 5 with TypeScript 5
- ESLint 9 + Prettier 3
- Zod for schema validation
- Helmet for security
- Morgan for logging

**Use Cases:**
- RESTful API development
- Microservices architecture
- Backend services for web/mobile apps
- Integration with legacy systems
- Projects requiring Node.js ecosystem compatibility

**Key Benefits:**
- Mature ecosystem with extensive packages
- Clean architecture (controllers, services, routes)
- Built-in security middleware
- Comprehensive error handling
- Wide community support

**Performance:** 40,000 requests/second

---

### 4. Bun + Elysia Boilerplate
**Repository:** [bun-boilerplatte](https://github.com/sprintx-official/bun-boilerplatte)

**Technology Stack:**
- Bun runtime with Elysia framework
- TypeScript 5 (strict mode)
- Biome for linting and formatting
- Auto-generated Swagger documentation
- Built-in validation

**Use Cases:**
- High-performance API services
- Microservices requiring fast startup
- Modern TypeScript-first projects
- Projects prioritizing developer experience
- New greenfield applications

**Key Benefits:**
- 6.5x faster than Node.js
- Native TypeScript support (no transpilation)
- 10x faster package installation
- All-in-one toolkit (runtime, package manager, bundler, test runner)
- End-to-end type safety
- Auto-generated API documentation

**Performance:** 260,000 requests/second

---

## Performance Comparison

| Metric | React | Next.js | Node.js + Express | Bun + Elysia |
|--------|-------|---------|-------------------|--------------|
| Runtime | Browser | Node.js | Node.js | Bun |
| Requests/sec | N/A | N/A | 40,000 | 260,000 |
| Package Install | ~5s | ~5s | ~8s | ~0.8s |
| Type Safety | Full | Full | Full | End-to-end |
| SSR Support | No | Yes | N/A | N/A |

---

## Shared Features

All boilerplates include:

- Strict TypeScript configuration
- Git hooks with Husky (pre-commit, commit-msg, pre-push)
- Conventional commit enforcement
- Branch name validation
- Automated code formatting
- Comprehensive linting rules
- Path aliases for clean imports
- Environment variable validation
- Professional documentation
- Production-ready architecture

---

## Selection Guide

**Choose React.js** when:
- Building client-side applications
- SEO is not critical
- Need fast client-side routing
- Want maximum UI flexibility

**Choose Next.js** when:
- SEO is critical
- Need server-side rendering
- Building full-stack applications
- Want optimal performance and Core Web Vitals
- Need static site generation

**Choose Node.js + Express** when:
- Building traditional REST APIs
- Need maximum npm package compatibility
- Working with legacy Node.js systems
- Team has Node.js expertise
- Require proven, stable technology

**Choose Bun + Elysia** when:
- Performance is top priority
- Starting a new project
- Want fastest development iteration
- Need type-safe API development
- Prefer modern tooling

---

## Getting Started

Each boilerplate includes:
- Detailed setup documentation in `docs/SETUP.md`
- Architecture overview in `docs/PROJECT.md`
- Contribution guidelines in `docs/CONTRIBUTING.md`
- Quick start guide in `QUICK_START.md`

Clone the desired repository and follow the setup instructions to begin development immediately.

---

## Quality Standards

All boilerplates enforce:
- Strict TypeScript compilation
- Pre-commit linting and formatting
- Conventional commit messages
- Type checking before push
- Production build validation

---

## Support

For issues, questions, or contributions, please refer to the individual repository's issue tracker and contribution guidelines.
