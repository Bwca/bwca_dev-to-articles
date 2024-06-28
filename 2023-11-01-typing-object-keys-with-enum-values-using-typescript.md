```ic-metadata
{
  "name": "Typing Object Keys With Enum Values Using Typescript",
  "series": null,
  "date": "2023-11-01",
  "lastModifiedDate": "2023-11-01",
  "author": "Volodymyr Yepishev",
  "tags": ["typescript", "generics", "tutorial"],
  "canonicalLink": "https://dev.to/bwca/typing-object-keys-with-enum-values-using-typescript-4k23"
}
```

# Typing Object Keys With Enum Values Using Typescript

Enums provide an excellent way to group constants in Typescript, but how to type objects when there is a need to use them to generate object keys? Today we will explore this.

We will discover how to perform dynamic typing for objects which utilize enum values as keys to point to a given value type. This can prove beneficial in a situation where one has an enum of features and wants to turn it into a dictionary to denote which features are enabled, and which are not, or if an enum of grades needs to be turned into a dictionary to show statistics how many of each grade students scored during a test.

## Crafting the functional interface

Foremost, it should be mentioned that there are two ways to create enumerations in Typescript, the first one being using `enum` keyword and the other one using a constant with `as const`. The latter has been discussed in this [article](https://dev.to/bwca/an-alternative-way-of-creating-enumerations-in-typescript-579k). The approach we are going to study is applicable to both. To make matters more interesting, let us assume the object manufactured from the `enum` might have only several of its values, not the whole set.

Therefore, we want to be able to pass an array of `enum` values and some other arguments to a function, provide a value type for the new mapped object's keys  and have the returned type based off this data.

This can be expressed via the following functional interface (fear not, detailed explanation under the code block):
```typescript
interface  Converter<A  extends  Array<unknown>, V> {
    <T>(keys: Array<T>, ...args: A): Record<string & T, V>
}
```
The generics here represent the following:

- `A` an array of additional values, passed to the converted function, left as an array of `unknown`, since the interface does not really care about them;
- `V` the type for the values, the mapped object keys will be pointing, this will provide the flexibility, we do not impose any restrictions;
- `T` this is the `enum` type, which will be passed.

Simple as that, this is all the heavy lifting that needs to be done, what is left now is to implement the interface for a couple of use cases.

## Case 1: Grading students
Consider the following situation: students have written a test and we want to collect data, how many occurrences of each grade there is in the test.

We will be using both types of enumerations for demo purposes:
```typescript
enum  Grades {
    a = 'A',
    b = 'B',
    c = 'C',
    d = 'D',
    e = 'E',
    f = 'F'
}

const  GRADES = {
    a: 'A',
    b: 'B',
    c: 'C',
    d: 'D',
    e: 'E',
    f: 'F'
} as  const;
```
The imaginary graded students come from an api and correspond to the following interface:
```typescript
interface  GradedStudent {
    name: string;
    grade: string;
}
```
Hence, let us implement grade checker function, we know that one of the extra arguments is going to be the array of graded students and we will be mapping to an integer, since we are counting grades:
```typescript
const gradeChecker: Converter<[Array<GradedStudent>], number> = (keys, students) => {
	const gradesStats: Record<string, number> = {};
	const keySet = new  Set(keys as  string[]);

	students.forEach(s => {
		const grade = s['grade'];
		if (keySet.has(grade as  string)) {
		gradesStats[grade] = (gradesStats[s['grade']] ?? 0) + 1;
		}
	});

	return gradesStats;
}
```
As some test data, let us also to generate an array of graded students
```typescript
const studentGrades: Array<GradedStudent> = Array.from({ length: 20 }, (_, index) => ({
	name: `Student ${index + 1}`,
	grade: (String.fromCharCode(65 + (index % 6)) as  unknown  as  typeof  GRADES)
}) as  unknown  as  GradedStudent);
```

As such, we can now turn an array of enum values into a dictionary of grade-count by passing an array of them into our `gradeChecker` function along with the mocked graded students:
```typescript
// for 'as const', resulting type 'Record<"A" | "B" | "C" | "D" | "E" | "F", number>'
const grades = gradeChecker(Object.values(GRADES), studentGrades);
console.log(grades);
// { "A": 4, "B": 4, "C": 3, "D": 3, "E": 3, "F": 3 }

// for 'enum', resulting type 'Record<Grades, number>'
const grades2 = gradeChecker(Object.values(Grades), studentGrades);
console.log(grades2);
// { "A": 4, "B": 4, "C": 3, "D": 3, "E": 3, "F": 3 }
```
What really makes this approach interesting, is that the resulting object type is based on the input, so it is dynamic, were we to pass only a single grade type, we would have gotten a different result, i.e. if we tried counting only `C` grades:
```typescript
// for 'as const', resulting type 'Record<"C", number>'
const cGrades = gradeChecker([GRADES.c], studentGrades);
console.log(cGrades);
// { "C": 3 }

// for 'enum', resulting type 'Record<Grades.c, number>'
const cGrades2 = gradeChecker<typeof  Grades.c>([Grades.c], studentGrades);
console.log(cGrades2);
// { "C": 3 }
```
Take note, how `as const` shines here, providing better typing experience compared to `enum`, which requires explicit passing of `typeof  Grades.c` to achieve same result.

## Case 2: Determining Feature Status

For this scenario, let us imagine that our application has a number of features, the name of which are stored in an enum, but requires an API call to fetch a configuration object, to determine which of them are enabled, and which not. Now we want to turn the list of features into a dictionary, where key is feature name and value is its enabled or disabled state, represented by a boolean.

We will use the following hypothetical feature enumerations (both `enum` and `as const` for demo purposes):
```typescript
enum  Features {
	SearchByName = 'searchByName',
	SearchBySurname = 'searchBySurname',
	SearchById = 'searchById',
}

const  FEATURES = {
	SearchByName: 'searchByName',
	SearchBySurname: 'searchBySurname',
	SearchById: 'searchById',
} as  const;
```

With the following feature status interface and a config object:
```typescript
interface  FeatureStatus {
	name: string;
	status: 'on' | 'off';
}

const  CONFIG: FeatureStatus[] = [
	{
		name: 'searchByName',
		status: 'on'
	},
	{
		name: 'searchBySurname',
		status: 'off'
	},
	{
		name: 'searchById',
		status: 'on'
	}
];
```
Implementing feature checker is going to be easier here, since we do not have any extra arguments to pass, we are assuming the config object is already available inside the checker function:
```typescript
const featureChecker: Converter<[], boolean> = (featureNames) => {
	const featuresStatus: Record<string, boolean> = {};
	const featuresSet = new  Map(CONFIG.map(f => ([f.name, f.status])));
	
	featureNames.forEach(name => {
		featuresStatus[name as  string] = featuresSet.get(name as  string) === 'on';
	});
	
	return featuresStatus;
}
```
Thus checking feature availability is as easy as passing the list of features to be checked:
```typescript
// for 'as const', resulting type 'Record<"searchByName" | "searchBySurname", boolean>'
const featuresAvailable = featureChecker([FEATURES.SearchBySurname, FEATURES.SearchByName]);
console.log(featuresAvailable);
// { "searchBySurname": false, "searchByName": true }
  
// for 'enum', resulting type 'Record<Features.SearchByName | Features.SearchBySurname, boolean>'
const featuresAvailable2 = featureChecker<Features.SearchBySurname | Features.SearchByName>([Features.SearchBySurname, Features.SearchByName]);
console.log(featuresAvailable2);
// { "searchBySurname": false, "searchByName": true }
```
Once again, note how neater `as const` is.

That concludes it, how you learned something. I know I did :)

P.S. the code is available in [the playground](https://www.typescriptlang.org/play?jsx=0#code/PTAEEkBtIVwZwC4CcCGCCWB7AdnUAzTJUACwQQAc4AuEAEwFMA3AOgU2ACMB3AYxWAIAnhXTYA5gFpMnAFYNeCSQGsGQuJO7oEJSQ2wwAtpKYpYDDfDFThFC7yToKSgCzKATAGYAUN+AAqUABhVHwMCVAdBgIYbEUsbDNQMQQGJHwUXmj-YF8UtIys4JwmNNSkAB4AQVAGAA9U7Do8KqRUIQrY5WxMbmwAPgAaUAA1ftAAb29QUAqAFX6AClV1alBW9vmh0BZdlCRxGnWASjWAJQUiOgrERwiAMlA54bHvAF9fP0CglDhogEY1gBxVB0aygRAwRjYBB4HK+fRGUAglCMPBTGYoUAAXlAAHIqnjBtNQJwcfiAEJEkm8cl4oLUmZ0OkAEUZtTpAFF2fg6QAxPHvXy8HCIZFnKoszkAZXJGNAKDWBPZnCVVOJM14SoZGtAdCVbN1DCV3N1+CVAveCrwItwCAA3HkYQVMtEUYw6NKEFD9AhJiTEoZjRDkNZHTNxKDg7cw0LhaK-ZHUQwgiQFKokGsgiUymkKgBtDYoDruhie73QhD9AC6wwMhk4aXGuOWajgw0hlbgxxx43ltrFSbRXrQRwuIqQ1xjEjrRkbSGbkzejppCdAK2lDD9uOwDG4oE3CFb6mtIbu4nz1eOK5mnd9cBYhCQnMyJEWeGxfZJmrXQ+iuLgfM8T-PFq3DGZkl5Y9DxYEhfkWP9T2ncRjh7eUINAP84BHWF8z-atyQQqNsIQUd80A4Co1AgiAH4aNAAAGHsAGpQH+cCZg+Tjr18GYkC3GAkGwTDiJwuBHQ+eM7RDH0YVLI4ixLKNy1kqtyUUx8kEwQxFgmUBIH0cQdDWdwGNAN5hkWAB9YYxEYOoe0-UBdIDFAgzWAADL1VNAAASCY7PqUBWP+N4PN1P81kWL1z007TU32bNGEWAA2ABWYLnMCupQAAUlAFLUNPLoej6U9bAYTBeSBCUpWlY53h7X5QBK3phOa0sVMrHjvAHRNiPJP9U3TNJFgAeTkBQEBYUxzDgRYaslGVjg7CtfXkniB0wAyWEgTBxCI5NuxXPqRKO9xBqjYbeAzcbJsUGazBgCwFuIlaZMrDbHS2na9oOrD3B607eHky7k2u2780WuqWF4WsPvWt7vtFbaGF2-bFhBpGpLFLHzrBxgIbzCqquRYjYaWKHybh1bVK+3qUd+jG8bRQGVy+YJfmidw1hZLc0kMMRwT5Bg0EE6IcPgUB4QRetQBFsX+PRElN32XgSApIQADk3P-fE-jVjXtd19lVaQdXNelQTAz1vEDfNo2raEk3dTNi2hHAZlcTt0WHc1z3qUk06+U5Ko5gAVTOGU5RV333Z19z9bjo2E4YU3k8t63daVe33adm308N-39STouPboQPTz6ld8nSV15dF71+Ml5WZhttZkPAxAxaOPEcDxUAAB98Sq-A8QknG-SCMatb5cAgTWBWm4YFvL3JfMSXQ0B29Lv3jaDdlb1I71e-7kkLM379t+z3e86zg-dSPnulVHwVON1Led59suA8fkNn5HtgN+5lvDVkngQRu4siaZmKNgUoSBygFnhpwTAqMUADEIvgSB-FU7dl7P6H80ksGKwsC3c4lxJw3FDDOUkqCDLoMXBMZcvFQCnWIcvbCW5yS7n3AAWRQBQRY09Z7zxYIYARixeROUWPmfALAbbDDkd3E+V5UI3ggSQ3Bj4iAvnVosG2+Ct7sPFiRHu+YDHNWQgRXExilYwXEFufRuskLUJQjibE3t+7gTeA1Ek-Em7CVsaQ4+8AJ4MyIdgiwVRTDoEgCgTgBlyRBOgTIkOYdI4yhYG7R298GDDDSRHKO0oskZ33gwK8yNcCo3RgdIJcBokoFifEgyvi2GRPqTEuJCSGAXRsZE6BFQl4mJKWXfOzjh5DKViMveqdKaTIsNMu+zsgz5PaYslOusKnhLgNUv6kj2kNKad0wGvggA).
