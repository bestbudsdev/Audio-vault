name: Transcribe Voice Memo with Gemini API
```

on:
  push:
    branches:
      - main # Or your main branch
    paths:
      - 'voice-memos/**' # Trigger when files are added/modified in the 'voice-memos' folder

jobs:
  transcribe:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.x'

      - name: Install Python Dependencies
        run: |
          pip install google-generativeai

      - name: Download Voice Memo File
        run: |
          # Assuming the voice memo file path is available in the event context
          # (You might need to adjust this based on how you upload files)
          FILE_PATH="${{ github.event.commits[0].added[0] }}" # Example - might need refinement
          echo "Downloading file: $FILE_PATH"
          # ... (Code to download the file - likely already checked out by actions/checkout)
          FILENAME=$(basename "$FILE_PATH")
          echo "FILENAME=$FILENAME" >> $GITHUB_ENV
          VOICE_MEMO_PATH="./$FILE_PATH" # Adjust if needed
          echo "VOICE_MEMO_PATH=$VOICE_MEMO_PATH" >> $GITHUB_ENV

      - name: Transcribe with Gemini API
        env:
          GOOGLE_API_KEY: ${{ secrets.GOOGLE_API_KEY }} # Store API key as a GitHub Secret
        run: |
          python <<EOF
          import google.generativeai as genai
          import os

          genai.configure(api_key=os.environ["GOOGLE_API_KEY"])
          model = genai.GenerativeModel('gemini-2.0-flash') # Or 'gemini-pro' if you have access and want to try it

          file_path = os.environ["VOICE_MEMO_PATH"]

          try:
              with open(file_path, 'rb') as f:
                  audio_data = f.read()

              audio_part = {"mime_type": "audio/mp3", "data": audio_data} # Adjust mime_type if needed (see supported formats in Gemini docs)

              response = model.generate_content(
                  contents=[
                      "Generate a transcript of the speech in this audio.", # Prompt for transcription
                      audio_part
                  ]
              )
              response.resolve() # Ensure response is fully resolved before accessing text

              transcribed_text = response.text
              print(f"Transcribed Text:\n{transcribed_text}")

              # Format as Markdown
              markdown_content = f"""---
              title: {os.environ["FILENAME"]} Transcription
              date: $(date +'%Y-%m-%d %H:%M:%S')
              ---

              ## Transcription:

              {transcribed_text}
              """

              with open(f"{os.environ['FILENAME']}.md", "w") as md_file:
                  md_file.write(markdown_content)

              echo "Markdown file created: {os.environ['FILENAME']}.md"

          except Exception as e:
              print(f"Error during transcription: {e}")
              exit(1) # Indicate failure in the workflow

          EOF

      - name: Commit Markdown File
        run: |
          git config --local user.email "actions@github.com"
          git config --local user.name "GitHub Actions"
          git add "*.md" # Add all Markdown files
          git commit -m "Add transcription for ${{ env.FILENAME }}" || echo "No changes to commit" # Only commit if there are changes
          git push origin main # Or your branch name
```

This needs 