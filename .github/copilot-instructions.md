# Copilot Instructions

## Project Guidelines
- In MyTaskAssistant repo: update MyTaskAssistant.html, specs/MyTaskAssistant_Specification.md, ReadMe.html, and ReadMe.md in place without creating version-suffixed filename copies (files are under source control); version numbers inside document contents may still be incremented.
- MyTaskAssistant is a single-page HTML application. Do not attempt a Visual Studio solution build unless the user specifically asks for it; validate with HTML/embedded JavaScript checks instead.

## AI Productivity Terminology
- Use "human-only time" only when Time Spent is greater than zero and AI Assistance is exactly 0%. 
- When AI Assistance is greater than 0%, describe the remainder as the "non-AI-assisted portion" of human-directed partnership work. 
- Avoid describing non-AI-assisted portions of recorded work as "human-only time". Treat work as a human-AI partnership; use neutral language such as "non-AI-assisted portion" or "remaining portion of recorded time".
- AI efficiency metrics must be derived from task source values, never by averaging task-level percentages. Use weighted efficiency: total estimated time saved divided by total AI-assisted hours. Support zero and negative time savings, completed/Closed task inclusion rules, and full-precision intermediate calculations.