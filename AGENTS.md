Rules
Rule 1: Each developer owns one module. Do not modify another developer's module without discussion.
Rule 2: Follow Contract First Development. Interfaces must be agreed before implementation.
Rule 3: Do NOT wait for another developer. If another module is unavailable, implement a Mock version first.
Rule 4: Depend on interfaces, not implementations. Use Dependency Injection whenever possible.
Rule 5: Raise blockers immediately.

Never use "any" type, create suitable interfaces/custom types for this project, and use them where and when needed. Do not skip any type errors, you need to resolve them.
Run npx run lint after finishing writing 1 sprint of code to check the code for any linting errors.
Run npx run build after finishing writing 1 sprint of code to check the code for any build errors.
Do not skip any error or warning from ESLint, figure a fix for it, and ask me if no fix is available for the error or warning.
If the values needed are in .env or have a enum type, use them instead of hardcoding values in the code.
Use @nestjs/swagger to properly document all routes.
Implement SOLID principles as you are a Senior SWE for every piece of code you write.
run "npm ci" after any changes to check for dependency errors.