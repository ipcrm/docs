Evaluate all your code according to your own standards. Code inspections let you locate problems and
measure how closely standards are followed. Run them on one repository or all repositories. Run them after every commit, so that developers are notified of the status of the code whenever they work in a repository.

## Installing an inspection from a Pack

{!tbd.md!}

## Custom Inspections

An inspection looks at a repository and produces some report. It is implemented as a function from Project to an inspection result, plus a separate function to react to these results. You decide what an inspection result contains, how to populate it, and how to react to them.

Create your inspection and a command to run it on demand in any project or projects. Then you can add it as an automatic inspection to every commit, if you like.

### Declare a result type

Start by deciding what your inspection wants to say about a repository. For instance, your inspection might look for files with too many lines. 
Your result might contain the paths of files that have too many lines in them.
Here, the type is defined as a string array.

```typescript 
export type FilesWithTooManyLines = string[];
```

### Create an inspection function

The CodeInspection is a function from a project (and optionally, inspection parameters) 
to an inspection result. The Project is an Atomist abstraction over a repository directory
and the files inside it. Your inspection can call functions on the Project to determine the result.

For instance, this one gathers all the file paths where the content is over 1000 lines:

```typescript
const InspectFileLengths: CodeInspection<FilesWithTooManyLines, NoParameters> =
    async (p: Project) => {
        // this sample code returns the paths to TypeScript files with over 1000 lines
        const longFiles = await gatherFromFiles(p, "**/*.ts", async f => {
            const c = await f.getContent();
            const lineCount = c.split("\n").length;
            if (lineCount > 1000) {
                return f.path;
            } else {
                return undefined;
            }
        });
        return longFiles.filter(path => path !== undefined);
    }
```

### Create a function to react to this result

Usually when you run a code inspection, you want to report back to yourself or your team what the results were. Since your inspection returns a custom type, you have to define what to do with it.

We need a function that reacts to the inspection results. It takes an input an array of CodeInspectionResult which includes information about the repository that was inspected and the results of the inspection.

For instance, the following reaction function sends a message containing the identifying information of the project and a summary of the results:

```typescript
async function onInspectionResults(
    results: CodeInspectionResult<FilesWithTooManyLines>[],
    inv: CommandListenerInvocation) {
    const message = results.map(r =>
        `${r.repoId.owner}/${r.repoId.repo} There are ${r.result.length} files with too many lines`)
        .join("\n");
    return inv.addressChannels(message);
}
```

### Create a command to run the inspection and react to it

Combine the inspection and the reaction into an object, a command registration. The `intent` is what you'll type to get Atomist to run the inspection.

```typescript
export const InspectFileLengthsCommand: CodeInspectionRegistration<FilesWithTooManyLines, NoParameters> = {
    name: "InspectFileLengths",
    description: "Files should be under 1000 lines",
    intent: "inspect file lengths",
    inspection: InspectFileLengths,
    onInspectionResults,
}
```

### Register the command on your SDM

Finally, teach the SDM about your command. In `machine.ts`, or 
wherever you configure your SDM, add 

```typescript
sdm.addCodeInspectionCommand(InspectFileLengthsCommand);
```

### Run the inspection

Recompile and restart your SDM. Depending on the context where you run `@atomist inspect file lengths`, you'll receive a response for one or many projects.

For local mode: run it within a repository directory to inspect one project, or one directory up (within an owner directory) to inspect all repositories under that owner, or anywhere else to inspect all repositories.

For team mode, in Slack: address Atomist in a channel linked to a repository to inspect that repository: `@atomist inspect file lengths`.
Or, specify a regular expression of repository names to check them all:`@atomist inspect file lengths targets.repos=".*"`.

## Create an AutoInspect

You may use your inspection to find places in the code that need to change, and then change them. But how will you know when the file lengths creep back up?

Make an AutoInspect run on every push. (Or in local mode, on every commit.) Then you can point out when a file has reached 1000 lines. You can point this out with a message, or by failing the goal, or by asking people to push a button to approve the unorthodox file length.

### Decide what should happen: onInspectionResult

What qualifies as a failed inspection? and what should happen when an inspection fails?

Decide this in a function from your inspection result type to a PushReactionResponse: "proceed", "fail", or "require approval".

You also get access to an invocation object, in case you want to post a message as well.

```typescript
async function failIfAnyFileIsTooLong(result: FilesWithTooManyLines,
    inv: ParametersInvocation<NoParameters>) {
    if (result.length === 0) {
        return PushReactionResponse.proceed;
    }
    await inv.addressChannels("The following files have more than 1000 lines:\n" + result.join("\n"));
    return PushReactionResponse.failGoals;
}
```

### AutoInspectRegistration

Assemble the inspection and the onInspectionResult into a registration:

```typescript
export const AutoInspectFileLengths: AutoInspectRegistration<FilesWithTooManyLines, NoParameters> = {
    name: "AutoInspectFileLengths",
    inspection: inspectFileLengths,
    onInspectionResult: failIfAnyFileIsTooLong,
}
```

### AutoInspect goal

Finally, register this on your AutoCodeInspection goal:

```typescript
    const codeInspection = new AutoCodeInspection();

    codeInspection.with(AutoInspectFileLengths);
```

Activate your AutoCodeInspection by setting the goal when a push happens. See [AutoInspect goal][autoinspect-goal] for details. 

[autoinspect-goal]: goal.md#autoinspect (AutoInspect Goal) 

