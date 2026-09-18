# mb-pipelines-openapi-generator

Pipeline to generate clients and mocks based on openapi specification using the openapi generator.
For more information:
- https://github.com/OpenAPITools/openapi-generator/tree/master
- https://github.com/OpenAPITools/openapi-generator/blob/master/docs/generators/java.md

## local usage
First download the jar using the [openapi readme](https://github.com/OpenAPITools/openapi-generator/tree/master?tab=readme-ov-file#13---download-jar).

Run all commands **from the repository root**: the generator config sets `templateDir`
as a relative path, so it is resolved against the working directory.

example commands:

### local debugging
```
java -jar openapi-generator-cli-7.15.0.jar generate -i supporting-files/oas-input/petstore-api.yaml -c supporting-files/generator-configs/java-jersey3.yaml -o ./target/generated-client
```

Use `java-jersey3-no-template.yaml` instead to generate with the stock jersey3
templates (no interface/Client split, no mock providers).

Output json model for operations to use in template files
```
java -jar openapi-generator-cli-7.15.0.jar generate -i supporting-files/oas-input/petstore-api.yaml -c supporting-files/generator-configs/java-jersey3.yaml -o ./target/generated-client --global-property debugOperations=true
```

### via docker (as the pipeline does it)
`-w /local` is required so that the relative `templateDir` in the config resolves
inside the mount; the image itself has no working directory set, so it defaults to `/`.
```
docker run --rm -v "$PWD:/local" -w /local openapitools/openapi-generator-cli:v7.15.0 generate -i /local/supporting-files/oas-input/petstore-api.yaml -c /local/supporting-files/generator-configs/java-jersey3.yaml -o /local/target/generated-client
```

### mock
```
java -jar openapi-generator-cli-7.15.0.jar generate -i oas-files/out-directapply-rone-v2.yaml -g java-wiremock -o ./output/out-directapply-rone-v2 --api-name-suffix apiClient --invoker-package nl.randstadgroep.xone.ta.client.directapply.rone.v2.client --api-package nl.randstadgroep.xone.ta.client.directapply.rone.v2.api --model-package nl.randstadgroep.xone.ta.client.directapply.rone.v2.model --additional-properties=asyncNative=false,groupId=nl.randstadgroep.xone.ta.client,artifactId=out-directapply-rone-api-client-v2,artifactVersion=2.0.0-ba680182-SNAPSHOT,licenseName=OwnedByRandstad,licenseUrl=https://randstadgroep.nl/,developerName=NLTestAutomation,developerEmail=xone.nl.test.automation@randstadgroep.nl,developerOrganization=randstadgroep,developerOrganizationUrl=https://randstadgroep.nl/,dateLibrary=java8,useJakartaEe=true,openApiNullable=false,serializableModel=true,serializationLibrary=jackson,generateClientAsBean=true,library=native,legacyDiscriminatorBehavior=true
```