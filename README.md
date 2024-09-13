# Text Rewriter Prototype

This project is a prototype for an interactive text rewriting application using a local LLM (Ollama with Gemma 2B) for real-time paragraph rewriting. It's built with Svelte and TipTap, providing a smooth, responsive user experience.

## Features

- Real-time text rewriting using Ollama's local Gemma 2B model
- Split-screen interface with editable input and dynamically rewritten output
- Paragraph-by-paragraph rewriting with visual feedback
- Resizable split-screen layout
- Debounced input processing to optimize performance

## Tech Stack

- Frontend: Svelte
- Text Editor: TipTap
- Local LLM: Ollama with Gemma 2B model
- Styling: Tailwind CSS

## How It Works

1. The user types or pastes text into the left-hand editor pane.
2. The input text is split into paragraphs.
3. Each paragraph is sent to the local Ollama server running the Gemma 2B model.
4. The model rewrites the paragraph, aiming for a more natural and engaging tone.
5. Rewritten paragraphs are displayed in real-time on the right-hand pane.

## Setup

1. Ensure you have [Ollama](https://ollama.ai/) installed and the Gemma 2B model pulled:
   ```
   ollama pull gemma2:2b
   ```

2. Clone this repository:
   ```
   git clone https://github.com/asmirbe/Text-Rewriter-Prototype.git
   cd text-rewriter-prototype
   ```

3. Install dependencies:
   ```
   npm install
   ```

4. Start the development server:
   ```
   npm run dev
   ```

5. Open your browser and navigate to `http://localhost:5000` (or the port specified by your Svelte setup).

## Usage

1. Type or paste your text into the left-hand "Editor" pane.
2. Watch as each paragraph is rewritten in real-time in the right-hand "Co-Pilot" pane.
3. Adjust the split-screen layout by dragging the divider between the panes.

## Limitations and Future Improvements

- Currently uses a fixed prompt for rewriting. Future versions could allow customizable rewriting styles.
- Error handling could be improved for more robust performance.
- The UI could be enhanced with additional features like saving rewrites, comparing versions, etc.

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## License

This project is open source and available under the [MIT License](LICENSE).