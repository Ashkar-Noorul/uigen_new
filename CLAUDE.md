# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

UIGen is an AI-powered React component generator built with Next.js 15. It allows users to describe React components in natural language, generates them using Claude AI, and provides a live preview in an isolated iframe environment. The application uses a virtual file system (no files written to disk) and supports both authenticated and anonymous users.

## Development Commands

### Setup
```bash
npm run setup
```
Installs dependencies, generates Prisma client, and runs database migrations.

### Development Server
```bash
npm run dev
```
Starts Next.js dev server with Turbopack at http://localhost:3000. Uses `node-compat.cjs` for Node.js compatibility.

### Testing
```bash
npm test         # Run all tests with Vitest
npm test -- --ui # Run tests with UI
```
Tests are located in `__tests__` directories alongside components (e.g., `src/components/chat/__tests__/`).

### Database
```bash
npx prisma generate           # Generate Prisma client
npx prisma migrate dev        # Create/apply migrations
npm run db:reset              # Reset database (force)
npx prisma studio             # Open Prisma Studio
```

Prisma client is generated to `src/generated/prisma/` (custom output path).

### Linting & Building
```bash
npm run lint     # Run ESLint
npm run build    # Production build
npm start        # Start production server
```

## Architecture

### Virtual File System

The core of UIGen is the `VirtualFileSystem` class (`src/lib/file-system.ts`) which manages an in-memory file tree. Key features:

- Files are stored in a `Map<string, FileNode>` structure
- Paths are normalized to start with `/`
- Supports standard operations: create, read, update, delete, rename
- Provides text editor commands (view, str_replace, insert) for AI tool use
- Can serialize/deserialize to JSON for database persistence

**Usage Pattern:**
```typescript
const fs = new VirtualFileSystem();
fs.createFile('/App.jsx', 'export default function App() {...}');
fs.readFile('/App.jsx');
fs.updateFile('/App.jsx', newContent);
```

### AI Tool System

The AI chat endpoint (`src/app/api/chat/route.ts`) provides Claude with two tools:

1. **`str_replace_editor`** (`src/lib/tools/str-replace.ts`): Text editor interface supporting:
   - `view`: Display file contents with line numbers
   - `create`: Create new files with parent directories
   - `str_replace`: Replace exact string matches
   - `insert`: Insert text at specific line numbers

2. **`file_manager`** (`src/lib/tools/file-manager.ts`): File operations:
   - `rename`: Move/rename files (creates parent dirs)
   - `delete`: Remove files/directories recursively

These tools are registered with the Vercel AI SDK's `streamText()` function.

### Preview System

The preview mechanism (`src/components/preview/PreviewFrame.tsx`) runs generated React components safely:

1. **JSX Transformation** (`src/lib/transform/jsx-transformer.ts`):
   - Converts JSX/TSX to JavaScript using Babel standalone
   - Creates ES modules as blob URLs
   - Generates an import map for browser-native ES modules
   - Resolves `@/` aliases to blob URLs

2. **Iframe Isolation**:
   - Sandboxed iframe with `allow-scripts`, `allow-same-origin`, `allow-forms`
   - Uses `srcdoc` with generated HTML containing import map
   - Entry point defaults to `/App.jsx`, fallback to other common entry files

### Provider System

The `getLanguageModel()` function (`src/lib/provider.ts`) implements a fallback pattern:

- **With API Key**: Uses `claude-haiku-4-5` via `@ai-sdk/anthropic`
- **Without API Key**: Returns `MockLanguageModel` that generates static demo components (Counter, ContactForm, Card)

The mock provider simulates streaming responses with tool calls across multiple steps to demonstrate the agentic workflow without requiring an API key.

### Authentication & Projects

- **Authentication** (`src/lib/auth.ts`): JWT-based sessions using `jose` library
- **Anonymous Work Tracking** (`src/lib/anon-work-tracker.ts`): Manages temporary projects for non-authenticated users
- **Middleware** (`src/middleware.ts`): Protects specific API routes (`/api/projects`, `/api/filesystem`)
- **Database Schema** (`prisma/schema.prisma`):
  - `User`: email, password (bcrypt hashed)
  - `Project`: name, messages (JSON), data (serialized file system), optional userId
  - Projects can exist without a user (anonymous mode)

### State Management

The application uses React Context for global state:

- **FileSystemContext** (`src/lib/contexts/file-system-context.tsx`): Manages VirtualFileSystem instance and provides file operations to components
- Changes to the file system trigger `refreshTrigger` updates to re-render preview

## Key Conventions

### File Organization
- **Actions**: Server actions in `src/actions/` (create-project, get-project, get-projects)
- **Components**: Organized by feature (`auth/`, `chat/`, `editor/`, `preview/`, `ui/`)
- **Tests**: Co-located in `__tests__/` directories
- **Lib**: Core utilities, contexts, prompts, tools, transform functions

### Import Paths
- Use `@/` alias for all internal imports (configured in `tsconfig.json`)
- Example: `import { VirtualFileSystem } from "@/lib/file-system"`

### Styling
- Tailwind CSS v4 with PostCSS
- UI components built with Radix UI primitives (`@radix-ui/*`)
- Component variants managed with `class-variance-authority`

### Code Comments
- Use comments sparingly. Only comment complex code.
- Code should be self-documenting through clear naming and structure

### Database Schema
- The database schema is defined in the `prisma/schema.prisma` file. Reference it anytime you need to understand the structure of data stored in the database.

### AI Generation Prompt
The system prompt (`src/lib/prompts/generation.tsx`) instructs Claude to:
- Always create `/App.jsx` as entry point
- Use Tailwind CSS for styling
- Import non-library files with `@/` alias
- Keep responses brief
- Operate on virtual file system root (`/`)

## Testing

Tests use Vitest with React Testing Library and jsdom:
- Configuration in `vitest.config.mts`
- Test files follow pattern `*.test.tsx` or `*.test.ts`
- Uses `@vitejs/plugin-react` and `vite-tsconfig-paths`

## Environment Variables

Required for full functionality:
```
ANTHROPIC_API_KEY=your-api-key-here
```

Optional - application runs with mock provider if not provided.
