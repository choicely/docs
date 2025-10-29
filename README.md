# Choicely Mobile SDK Documentation

Official documentation for the Choicely Mobile SDK for Android and iOS. This documentation site is built with Mintlify and contains comprehensive guides, API references, and setup instructions for integrating Choicely into your mobile applications.

## Contents

This documentation includes:

- **Android SDK** - Complete guides for Java and Kotlin integration
- **iOS SDK** - Swift and UIKit implementation guides
- **Feature Guides** - Firebase, Maps, In-App Purchases, Ads, Articles, and more
- **Advanced Customization** - Custom UI components and advanced implementations
- **Setup Resources** - Third-party service configuration guides

## Migration Notes

This documentation was migrated from the legacy HTML-based documentation system to Mintlify on October 29, 2024. All content from the previous mobile-sdk-documentation repository has been converted to MDX format with enhanced navigation and Mintlify components.

## Development

Install the [Mintlify CLI](https://www.npmjs.com/package/mint) to preview your documentation changes locally. To install, use the following command:

```
npm i -g mint
```

Run the following command at the root of your documentation, where your `docs.json` is located:

```
mint dev
```

View your local preview at `http://localhost:3000`.

## Publishing changes

Install our GitHub app from your [dashboard](https://dashboard.mintlify.com/settings/organization/github-app) to propagate changes from your repo to your deployment. Changes are deployed to production automatically after pushing to the default branch.

## Need help?

### Troubleshooting

- If your dev environment isn't running: Run `mint update` to ensure you have the most recent version of the CLI.
- If a page loads as a 404: Make sure you are running in a folder with a valid `docs.json`.

### Resources
- [Mintlify documentation](https://mintlify.com/docs)
