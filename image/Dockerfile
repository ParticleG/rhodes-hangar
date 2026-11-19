# Android 14 (redroid) with the AVD 34 ARM translation layer, safe to run
# privileged: it never writes host-wide sysctls or tracing. `hangar deploy`
# builds the `avd34` target; it downloads ndk34.tgz into this directory first.
ARG REDROID
FROM ${REDROID} AS upstream
FROM busybox@sha256:a7dbc319b1438115a207e11930dc4bb3e643dcb3ec0bdd694bb36cd8fad289ec AS tools
FROM tools AS protect
COPY --from=upstream /system/etc/init/hw/init.rc /init.rc
# A privileged guest shares the host's module helper. Never clear it at boot.
RUN awk '$1 == "write" && $2 == "/proc/sys/kernel/modprobe" { removed++; next } { print } END { if (removed != 1) exit 1 }' /init.rc > /protected.rc

FROM upstream AS protected
COPY --from=protect /protected.rc /system/etc/init/hw/init.rc

FROM tools AS bridge
# ER5TT/ndk, commit 9ecefe66bb78e3347bbac7d2237befebd0552c15 (Google AVD 34 ext12 r01).
COPY ndk34.tgz /ndk34.tgz
RUN echo 'c18361e92be742d2b2e21f075081ca7cba97fbb4d251f8e3ff7bf04ad9bb1f65  /ndk34.tgz' | sha256sum -c - && \
    mkdir /payload && tar -xzf /ndk34.tgz -C /payload --strip-components=1 && \
    rm /payload/system/lib64/libalarm_jni.so
# Keep the parallel test instance from changing host-wide sysctls and tracing.
COPY --from=protect /protected.rc /rc/system/etc/init/hw/init.rc
COPY --from=upstream /system/etc/init/atrace.rc /system/etc/init/atrace_userdebug.rc /system/etc/init/init-debug.rc /system/etc/init/llkd.rc /rc/system/etc/init/
COPY --from=upstream /vendor/etc/init/redroid.common.rc /rc/vendor/etc/init/redroid.common.rc
RUN for f in /rc/system/etc/init/hw/init.rc /rc/system/etc/init/*.rc /rc/vendor/etc/init/*.rc; do \
      awk '$1 == "write" && $2 ~ /^\/proc\/sys\/(kernel|vm|abi)\// { next } $1 == "mount" && ($2 == "tracefs" || $2 == "debugfs") { next } /^[^#]*\/(sys\/kernel\/(debug|tracing)|sys\/power\/|sys\/module\/|proc\/sysrq-trigger|extra_free_kbytes\.sh)/ && $1 != "file" { next } { print }' "$f" > "$f.new" && mv "$f.new" "$f" || exit 1; \
    done

FROM protected AS avd34
COPY --from=bridge /payload/system/ /system/
COPY --from=bridge /rc/ /
