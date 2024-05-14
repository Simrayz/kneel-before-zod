---
theme: ./
---

# Kneel Before Zod

<div class="flex items-center gap-10">
    <img src="https://zod.dev/logo.svg" class="w-32" />
    <p>Runtime type-checking with TypeScript</p>
</div>


<div class="pt-12">
  <span @click="next" class="px-2 p-1 rounded cursor-pointer hover:bg-white hover:bg-opacity-10">
    Press Space for next page <carbon:arrow-right class="inline"/>
  </span>
</div>

---
---

# What is Zod?

TypeScript-first schema validation with static type inference

* Define schemas with zod and infer the type from it, <br/> `type SomeType = z.infer<typeof someSchema>`
* Schemas can contain strings, check formatting, enums etc.
    * `z.string().email()`
    * `z.enum(['apple', 'orange', 'banana'])`
* Schemas can also fetch async data, e.g. if you have a uniqueness check in a form.
    * `const usernameSchemaz.string().refine(async (username) => api.checkUniqueUsername(username))`
* Integrates with a lot of libraries to provide a uniform API for runtime validation. E.g. `react-hook-form`, `@mantine/form`, `zodios`, `trpc`. 
* Alternative to Joi and Yup, except Zod is 100% typed and really performant.

---

# Example: User schema with validation method

```ts
const userSchema = z.object({
  id: z.string().uuid(),
  name: z.string(),
  email: z.string().email(),
  role: z.enum(["admin", "user"]),
})

type User = z.infer<typeof userSchema>;

// Generic function to validate some random JSON from the API
export function parseIfValid<S extends ZodSchema>(
  schema: S,
  data: Record<string, unknown>
) {
  const parsed = schema.safeParse(data); // safeParse returns a result with .success or .error
  if (!parsed.success) return null; // Return null if validation fails
  return data as z.infer<typeof schema>; // Guaranteed to satisfy the schema type
}
// This call guarantees that a non-falsy result satisfies the User type
const validatedUser = parseIfValid(userSchema, {...}) 
```

---

# Usecase: Validate Deeply Nested Object

```ts
const roleSchema = z.object({
    id: z.string(),
    name: z.enum(["admin", "member"]),
})
const addressSchema = z.object({
    addressLine: z.string(),
    zipCode: z.string(),
    city: z.string(),
})
const userSchema = z.object({
    id: z.string().uuid({ message: "ID must be a valid UUID" }), // Custom error message
    name: z.string().nullable(), // string | null, but has to be included in the object.
    displayName: z.string().optional(), // string | undefined, can be omitted.
    email: z.string().email().endsWith(".no", { message: "Only .no email domains allowed" } );, // Custom error
    role: roleSchema,
    address: addressSchema,
})
type User = z.infer<typeof userSchema>;
// Guaranteed to satisfy the User type, or else it throws a validation error.
const user = userSchema.parse({...})
```

---

# Usecase: Validate Environment Variables

Example from VCC Mobile App

```ts
const envSchema = z.object({
  TENANT_ID: z.string(),
  CLIENT_ID: z.string(),
  API_CLIENT_ID: z.string(),
  API_URL: z.string(),
  IS_DEV: z.boolean(),
});

export type EnvClient = z.infer<typeof envSchema>;

const rawEnvVariables = {
  TENANT_ID: process.env.EXPO_PUBLIC_TENANT_ID,
  CLIENT_ID: process.env.EXPO_PUBLIC_CLIENT_ID,
  API_CLIENT_ID: process.env.EXPO_PUBLIC_API_CLIENT_ID,
  API_URL: process.env.EXPO_PUBLIC_API_URL,
  IS_DEV: process.env.NODE_ENV === 'development',
};

// Guarantees that all env variables are present, or else the app will crash.
export const env = envSchema.parse(rawEnvVariables);
```

---

# Usecase: Parsing WebSocket Notifications

The schema makes sure that we only pass known notifications to the handler.
```ts
export function handleAssistanceEvent(notification: BaseNotification, queryClient: QueryClient) {
  const assistanceEvent = parseIfValid(assistanceNotificationSchema, notification);
  if (!assistanceEvent) {
    console.error('Could not parse assistance event', notification);
    return null;
  }
  executeHandler(assistanceEvent, queryClient);
}
```

The schema checks that the event has a specific type ('assistance') and that it is one of the allowed events.

```ts
z.object({
    ...baseNotificationSchema,
    type: z.literal('assistance'),
    event: z.enum(ASSISTANCE_NOTIFICATION_EVENTS),
    ...restOfAssistanceNotificationSchema
})
```

---

# Usecase: Validating the Backend API with Zodios

```ts
// api/schemas/user.ts
export const userInfoSchema = z.object({
  name: z.string(),
  email: z.string(),
  roles: z.array(z.string()),
});
// api/client.ts
import { makeApi } from '@zodios/core';
/* Generates a typed client with `api.aboutMe`.
The endpoint will fail if the API introduces breaking changes in the User payload. 
This will only occur if existing fields are changed to an incompatible value. */
const api = makeApi({
    ...,
    {
        method: 'get',
        path: '/mobile/about-me',
        alias: 'aboutMe',
        description: `Information about users, extracted from the provided jwt token.`,
        requestFormat: 'json',
        response: userInfoSchema,
    },
    ...
})
```

---

# Usecase: Validating Forms

You can integrate zod schemas with form libraries such as `react-hook-form` and `@mantine/form`

```ts
const editOrganizationSchema = z.object({
  name: nameSchema,
  status: z.enum(ORGANIZATION_STATUSES).default('active').catch('active'), // default value is 'active', even if undefined in payload
  hasCarenotes: z.boolean().default(false),
});

type EditOrganizationValues = z.infer<typeof editOrganizationSchema>;

export function EditOrganizationForm({ organization, onClose }: EditOrganizationFormProps) {
  const form = useForm<EditOrganizationValues>({
    initialValues: editOrganizationSchema.parse(organization),
    validateInputOnChange: true,
    // The zodResolver converst the zod schema to match the interal form library validation logic.
    validate: zodResolver(editOrganizationSchema),
  });
  ... // render form here
}
```

---

# Conclusion 
<Youtube id="_xUiCNlDeJk?start=80" width="80%" class="aspect-video rounded-md"  />

