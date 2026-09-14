# scotland-tours

Personal repo for Ian's Scotland driving holiday. It holds self-guided walking tour pages,
generated on the phone and served by GitHub Pages.

- Any request for a walking tour, audio tour or guided walk: use the **walking-tour** skill at
  `.claude/skills/walking-tour/SKILL.md`. Do not improvise a page by hand.
- Tours live in `tours/<slug>.html` and are served at
  `https://ronangraphics.github.io/scotland-tours/tours/<slug>.html`. The index is at
  `https://ronangraphics.github.io/scotland-tours/`.
- Always `git push` after generating a tour, or the phone cannot see it. Pages takes about a minute.
- Never commit personal data: no home or hotel addresses, no booking references, no itinerary,
  no car registration, no phone numbers.
- Plain files only. No frameworks, no build step, no npm.
