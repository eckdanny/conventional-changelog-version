#!/usr/bin/env node

// @todo(de): just use npm for everything
// @todo(de): Oh nos! Am i reinventing [conventional-changelog/standard-version](https://github.com/conventional-changelog/standard-version)?

const { promisify } = require('util')
const { basename, dirname, join } = require('path')
const { inc } = require('semver')
const chalk = require('chalk')
const logger = require('npmlog')
const execA = require('execa')
const conventionalRecommendedBump = require('conventional-recommended-bump')

const recBump = promisify(conventionalRecommendedBump)
const scriptName = basename(__filename)

if (process.env.npm_lifecycle_event !== scriptName) {
	logger.error(scriptName, `only run \`${scriptName}\` as an npm lifecycle script please!`)
	process.exit(1)
}

(async () => {
	try {
		const { recommendedBumpOpts } = await require('conventional-changelog-angular-bitbucket')
		// @todo(de): get `whatBump` from preset dynamically
		const whatBumpAngularBitbucket = recommendedBumpOpts.whatBump
		const whatBump = (...args) => {
			const res = whatBumpAngularBitbucket(...args)
			warnIfNoUsefulCommits(res, ...args)
			return res
		}
		const bump = await recBump({ ...recommendedBumpOpts, whatBump })
		const nextVersion = inc(process.env.npm_package_version, bump.releaseType)
		const npmExec = process.env.npm_execpath

		logger.verbose(scriptName, { bump, npmExec, nextVersion })
		await execA(npmExec, [
			'version',
			'--silent',
			'--non-interactive',
			'--no-git-tag-version',
			'--new-version',
			nextVersion
		])
		// @todo(de): get dynamically (from argv?)
		await execA(getExecPath(), [
			'conventional-changelog',
			'-p', 'angular-bitbucket',
			'-i', 'CHANGELOG.md',
			'-s'
		])
		await execA('git', ['add', 'CHANGELOG.md', 'package.json'])
		await execA('git', ['commit', '-m', `chore(release): ${nextVersion}`])

		// @todo(de): get this from npm config or from preset config
		await execA('git', ['tag', '-a', '-m', `chore(release): ${nextVersion}`, `v${nextVersion}`])
	} catch (err) {
		logger.error(scriptName, err.message)
		process.exit(1)
	}
})()

function getExecPath() {
	const execPath = basename(process.env.npm_execpath)
	switch (true) {
		case (execPath === 'npm-cli.js'): {
			return join(dirname(process.env.npm_execpath), 'npx-cli.js')
		}
		case (execPath === 'yarn.js'): {
			return process.env.npm_execpath
		}
		default: {
			// @todo(de): better error msg'ing plz
			throw Error('nuts!')
		}
	}
}

function warnIfNoUsefulCommits(result, ...args) {
	const [commits] = args
	const LEVELS = ['major', 'minor', 'patch']

	// @todo(de): how can this be retrieved programatically from preset?
	const ACTION_TYPES = ['feat', 'fix', 'perf']

	const meaningfulCommits = commits.filter(commit => ACTION_TYPES.includes(commit.type))

	if (meaningfulCommits.length) return

	const msgs = []

	if (!commits.length) msgs.push('no commits since last release!')
	if (commits.length && !meaningfulCommits.length) msgs.push('no meaningful commits since last release!')

	msgs.push('an empty changelog section will be geneated')
	msgs.push(`using semver bump: ${chalk.yellow(LEVELS[result.level])}`)

	msgs.forEach(msg => logger.warn(scriptName, msg))
}
