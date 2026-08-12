# Contributing

Corrections and additions are welcome, as a pull request or an issue. This is a documentation repository, not a code project, so the bar is low: a typo fix, a broken link, a clarified sentence, or a new gotcha you hit in the field are all fair game.

## What this guide is

The guide is written for partner integration engineers: vendors adapting host RPM software for customers who are adopting RHEL image mode (bootc). It assumes that lens throughout. If a contribution reads like it's written for someone building their own bootc image from scratch, it's probably a better fit for the upstream bootc docs or the RHEL documentation than for this guide.

## Technical claims need a source

This guide gets read as authoritative, so every technical claim needs to trace back to something. When you propose a change, point to one of:

- Upstream [bootc documentation](https://github.com/bootc-dev/bootc/tree/main/docs)
- [RHEL documentation](https://docs.redhat.com/)
- The documentation of the specific project or tool you're describing

If you hit a wall the guide doesn't cover and you're not sure it belongs, open an issue and describe what you found. Someone can help place it.

## Voice

This guide follows a consistent voice. Please match it:

- No em dashes. Use commas, colons, or separate sentences instead.
- Don't use the word "shape." Say "layout," "structure," "pattern," or name the thing directly.
- Don't use the word "immutable." Say "image-based" for the deployment model, or "read-only" for filesystem state.
- Direct and simple. No marketing language.

## Making a change

1. Open a pull request against `main`, or open an issue if you'd rather flag something than write the fix yourself.
2. Keep changes focused. One correction or one addition per PR is easier to review than a bundle.
3. If you're adding a new pattern to the catalog, follow the existing structure: what fails, why, and what to do instead.

That's it. This isn't a kernel subsystem, no need for a formal RFC process.
