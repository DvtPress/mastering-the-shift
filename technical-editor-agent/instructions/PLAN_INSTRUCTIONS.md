# Technical Editing Agent Claude-Flow Instructions

> Commands issued
```
npx claude-flow swarm "Create implementation plan for application described in APP_REQUIREMENTS.md in folder instructions. Place the implementation plan in folder plans. Do not implement the feature additions yet. Please let me know if you need additional information." --claude
```

> Content of ```APP_REQUIREMENTS.md```
Create an AI agent that allows a user to supply folder with book content and have it fact checked and the text edited for clarity, accuracy, consistency, usability for readers.
- Book content input is Leanpub markdown format (https://leanpub.com/lfm/read)
- Allow the user to specify use of either the OpenAI API (version gpt-5) or Claude API (Claude Sonnet 4)
- Edited book output should overwrite the input format
- Preserve the markdown file structure and filenames as that represents chapters
- Any statement deemed inaccurate should generate a warning, but not stop the editing process. Don't bother reporting statements that are accurate.
- The application will change the wording of text as appropriate for clarity, accuracy, consistency, and usability
- The application will move text as appropriate for clarity, accuracy, consistency, and usability
- The application may move, but not change any content in markdown code blocks (text inbetween "```" symbols)
- The application may move, but not change content of any web page links (denoted by "[<label>](<web page>)") or graphics inserts (denoted by "![Image Description](filename)").
- Verify any broken web page links.
- Implement unit testing with mocks for API calls targeting 80% code coverage.
- Implement a github workflow that will perform an integration test using a test folder in the repo. You can assume that any API keys needed will be provided as GihHub secrets.

