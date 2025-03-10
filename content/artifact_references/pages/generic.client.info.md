---
title: Generic.Client.Info
hidden: true
tags: [Client Artifact]
---

Collect basic information about the client.

This artifact is collected when any new client is enrolled into the
system. Velociraptor will watch for this artifact and populate its
internal indexes from this artifact as well.

You can edit this artifact to enhance the client's interrogation
information as required, by adding new sources.

NOTE: Do not modify the BasicInformation source since it is used to
interrogate the clients.


<pre><code class="language-yaml">
name: Generic.Client.Info
description: |
  Collect basic information about the client.

  This artifact is collected when any new client is enrolled into the
  system. Velociraptor will watch for this artifact and populate its
  internal indexes from this artifact as well.

  You can edit this artifact to enhance the client's interrogation
  information as required, by adding new sources.

  NOTE: Do not modify the BasicInformation source since it is used to
  interrogate the clients.

sources:
  - name: BasicInformation
    description: |
      This source is used internally to populate agent info. Do not
      modify or remove this query.
    query: |
        LET Interfaces = SELECT format(format='%02x:%02x:%02x:%02x:%02x:%02x',
            args=HardwareAddr) AS MAC
        FROM interfaces()
        WHERE HardwareAddr

        SELECT config.Version.Name AS Name,
               config.Version.BuildTime as BuildTime,
               config.Version.Version as Version,
               config.Version.ci_build_url AS build_url,
               config.Version.install_time as install_time,
               config.Labels AS Labels,
               Hostname, OS, Architecture,
               Platform, PlatformVersion, KernelVersion, Fqdn,
               Interfaces.MAC AS MACAddresses
        FROM info()

  - name: WindowsInfo
    description: Windows specific information about the host
    precondition: SELECT OS From info() where OS = 'windows'
    query: |
      LET DomainLookup &lt;= dict(
         `0`='Standalone Workstation',
         `1`='Member Workstation',
         `2`='Standalone Server',
         `3`='Member Server',
         `4`='Backup Domain Controller',
         `5`='Primary Domain Controller')

      SELECT
          {
            SELECT DNSHostName, Name, Domain, TotalPhysicalMemory,
                   get(item=DomainLookup,
                       field=str(str=DomainRole), default="Unknown") AS DomainRole
            FROM wmi(
               query='SELECT * FROM win32_computersystem')
          } AS `Computer Info`,
          {
            SELECT Caption,
               join(array=IPAddress, sep=", ") AS IPAddresses,
               join(array=IPSubnet, sep=", ") AS IPSubnet,
               MACAddress,
               join(array=DefaultIPGateway, sep=", ") AS DefaultIPGateway,
               DNSHostName,
               join(array=DNSServerSearchOrder, sep=", ") AS DNSServerSearchOrder
            FROM wmi(
               query="SELECT * from Win32_NetworkAdapterConfiguration" )
            WHERE IPAddress
          } AS `Network Info`
      FROM scope()

    notebook:
      - type: vql_suggestion
        name: "Enumerate Domain Roles"
        template: |
          /*
          # Enumerate Domain Roles

          Search all clients' enrollment information for their domain roles.
          */
          --
          -- Remove the below comments to label Domain Controllers
          SELECT *--, label(client_id=client_id, labels="DomainController", op="set") AS Label
          FROM foreach(row={
             SELECT * FROM clients()
          }, query={
              SELECT
                `Computer Info`.Name AS Name, client_id,
                `Computer Info`.DomainRole AS DomainRole
              FROM source(client_id=client_id,
                  flow_id=last_interrogate_flow_id,
                  artifact="Generic.Client.Info/WindowsInfo")
          })
          -- WHERE DomainRole =~ "Controller"

  - name: Users
    precondition: SELECT OS From info() where OS = 'windows'
    query: |
      SELECT Name, Description, Mtime AS LastLogin
      FROM Artifact.Windows.Sys.Users()

reports:
  - type: CLIENT
    template: |
      {{ $client_info := Query "SELECT * FROM clients(client_id=ClientId) LIMIT 1" | Expand }}

      {{ $flow_id := Query "SELECT timestamp(epoch=active_time / 1000000) AS Timestamp FROM flows(client_id=ClientId, flow_id=FlowId)" | Expand }}

      # {{ Get $client_info "0.os_info.fqdn" }} ( {{ Get $client_info "0.client_id" }} ) @ {{ Get $flow_id "0.Timestamp" }}

      {{ Query "SELECT * FROM source(source='BasicInformation')" | Table }}

      # Memory and CPU footprint over the past 24 hours

      {{ define "resources" }}
       SELECT * FROM sample(
         n=4,
         query={
           SELECT Timestamp, rate(x=CPU, y=Timestamp) * 100 As CPUPercent,
                  RSS / 1000000 AS MemoryUse
           FROM source(artifact="Generic.Client.Stats",
                       client_id=ClientId,
                       start_time=now() - 86400)
           WHERE CPUPercent &gt;= 0
         })
      {{ end }}

      &lt;div&gt;
      {{ Query "resources" | LineChart "xaxis_mode" "time" "RSS.yaxis" 2 }}
      &lt;/div&gt;

      {{ $windows_info := Query "SELECT * FROM source(source='WindowsInfo')" }}
      {{ if $windows_info }}
      # Windows agent information
        {{ $windows_info | Table }}
      {{ end }}

      # Active Users
      {{ Query "SELECT * FROM source(source='Users')" | Table }}


column_types:
  - name: BuildTime
    type: timestamp
  - name: LastLogin
    type: timestamp

</code></pre>

