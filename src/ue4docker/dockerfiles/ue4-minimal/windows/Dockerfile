# escape=`
{% if combine %}
FROM source as builder
{% else %}
ARG NAMESPACE
ARG TAG
ARG PREREQS_TAG
FROM ${NAMESPACE}/ue4-source:${TAG}-${PREREQS_TAG} AS builder
{% endif %}

# Remove the .git directory to disable UBT `git status` calls and speed up the build process
RUN if exist C:\UnrealEngine\.git rmdir /s /q C:\UnrealEngine\.git

# Create an Installed Build of the Engine
WORKDIR C:\UnrealEngine
RUN .\Engine\Build\BatchFiles\RunUAT.bat BuildGraph `
	-target="Make Installed Build Win64" `
	-script=Engine/Build/InstalledEngineBuild.xml `
	-set:HostPlatformOnly=true `
	-set:WithClient=true `
	-set:WithDDC={% if excluded_components.ddc == true %}false{% else %}true{% endif %} `
	-set:WithServer=true `
	{{ buildgraph_args }} && `
	(if exist C:\UnrealEngine\LocalBuilds\InstalledDDC rmdir /s /q C:\UnrealEngine\LocalBuilds\InstalledDDC) && `
	rmdir /s /q C:\UnrealEngine\Engine

# Split out components (DDC, debug symbols, template projects) so they can be copied into the final container image as separate filesystem layers
COPY split-components.py C:\split-components.py
RUN python C:\split-components.py C:\UnrealEngine\LocalBuilds\Engine\Windows C:\UnrealEngine\Components

# Copy the Installed Build into a clean image, discarding the source tree
{% if combine %}
FROM prerequisites as minimal
{% else %}
ARG NAMESPACE
FROM ${NAMESPACE}/ue4-build-prerequisites:${PREREQS_TAG}
{% endif %}

# Copy the Installed Build files from the builder image
COPY --from=builder C:\UnrealEngine\LocalBuilds\Engine\Windows C:\UnrealEngine
COPY --from=builder C:\UnrealEngine\Components\Binaries C:\UnrealEngine
COPY --from=builder C:\UnrealEngine\Components\Content C:\UnrealEngine
COPY --from=builder C:\UnrealEngine\Components\Extras C:\UnrealEngine
COPY --from=builder C:\UnrealEngine\Components\Intermediate C:\UnrealEngine
COPY --from=builder C:\UnrealEngine\Components\Plugins C:\UnrealEngine
COPY --from=builder C:\UnrealEngine\Components\Source C:\UnrealEngine
{% if excluded_components.ddc == false %}
COPY --from=builder C:\UnrealEngine\Components\DDC C:\UnrealEngine
{% endif %}
{% if excluded_components.debug == false %}
COPY --from=builder C:\UnrealEngine\Components\DebugSymbols C:\UnrealEngine
{% endif %}
{% if excluded_components.templates == false %}
COPY --from=builder C:\UnrealEngine\Components\TemplatesAndSamples C:\UnrealEngine
{% endif %}
WORKDIR C:\UnrealEngine

{% if not disable_labels %}
# Add labels to the built image to identify which components (if any) were excluded from the build that it contains
LABEL com.adamrehn.ue4-docker.excluded.ddc={% if excluded_components.ddc == true %}1{% else %}0{% endif %} 
LABEL com.adamrehn.ue4-docker.excluded.debug={% if excluded_components.debug == true %}1{% else %}0{% endif %} 
LABEL com.adamrehn.ue4-docker.excluded.templates={% if excluded_components.templates == true %}1{% else %}0{% endif %} 
{% endif %}
