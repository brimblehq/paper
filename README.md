# Brimble Documentation

Source for the Brimble docs site.

## Local development

```bash
npm i -g mintlify
mintlify dev
```

That serves the site on `http://localhost:3000`.

## Editing

Pages live as `.mdx` files under directories that mirror the navigation in [`docs.json`](./docs.json). When you add a new page, register it under the appropriate `group` in `docs.json`'s `navigation.tabs[0].groups`.

Components available without imports: `<Note>`, `<Tip>`, `<Info>`, `<Warning>`, `<Check>`, `<Card>`, `<CardGroup>`, `<Tabs>`, `<Tab>`, `<Steps>`, `<Step>`, `<CodeGroup>`, `<Frame>`, `<Accordion>`, `<AccordionGroup>`. See [Mintlify components](https://mintlify.com/docs/content/components) for the full list.

## Conventions

- Every page has YAML frontmatter with `title` and `description`. Mintlify renders `title` as the H1, so don't write a duplicate `# Title` in the body.
- Internal links use Mintlify's absolute paths without the `.mdx` extension: `[Builds](/projects/builds)`.
- Image placeholders use `<Info>` blocks tagged `**Image needed:**` with descriptive alt text.
- Brand icons in the frameworks page come from [Simple Icons](https://simpleicons.org).

## Help

Issues with the docs themselves: [open an issue](https://github.com/brimblehq/paper/issues).
