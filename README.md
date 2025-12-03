<mxfile host="app.diagrams.net" modified="2025-12-03T00:00:00.000Z" agent="ChatGPT" version="22.1.0">
  <diagram id="branching-release-process" name="Branching + Release + ServiceNow Gate">
    <mxGraphModel dx="1200" dy="800" grid="1" gridSize="10" guides="1" tooltips="1" connect="1" arrows="1" fold="1" page="1" pageScale="1" pageWidth="1400" pageHeight="900" math="0" shadow="0">
      <root>
        <mxCell id="0"/>
        <mxCell id="1" parent="0"/>

        <!-- Title -->
        <mxCell id="t1" value="GitHub Branching Strategy + Tag-based Promotion (dev / preprod / prod) + ServiceNow Gate" style="text;html=1;strokeColor=none;fillColor=none;align=left;verticalAlign=top;fontSize=18;fontStyle=1" vertex="1" parent="1">
          <mxGeometry x="40" y="20" width="1200" height="30" as="geometry"/>
        </mxCell>

        <!-- Swimlane headers -->
        <mxCell id="h_dev" value="Developer" style="rounded=1;whiteSpace=wrap;html=1;strokeWidth=1;fillColor=#F5F5F5;strokeColor=#D0D0D0;align=center;fontStyle=1" vertex="1" parent="1">
          <mxGeometry x="40" y="70" width="320" height="50" as="geometry"/>
        </mxCell>
        <mxCell id="h_pr" value="PR Governance (Rulesets / Checks)" style="rounded=1;whiteSpace=wrap;html=1;strokeWidth=1;fillColor=#F5F5F5;strokeColor=#D0D0D0;align=center;fontStyle=1" vertex="1" parent="1">
          <mxGeometry x="380" y="70" width="360" height="50" as="geometry"/>
        </mxCell>
        <mxCell id="h_cd" value="CI/CD + Environments" style="rounded=1;whiteSpace=wrap;html=1;strokeWidth=1;fillColor=#F5F5F5;strokeColor=#D0D0D0;align=center;fontStyle=1" vertex="1" parent="1">
          <mxGeometry x="760" y="70" width="600" height="50" as="geometry"/>
        </mxCell>

        <!-- Developer lane nodes -->
        <mxCell id="n1" value="Create branch&#xa;feature/* or bugfix/*" style="rounded=1;whiteSpace=wrap;html=1;strokeWidth=1;fillColor=#E8F0FE;strokeColor=#6C8EBF" vertex="1" parent="1">
          <mxGeometry x="60" y="150" width="280" height="70" as="geometry"/>
        </mxCell>

        <mxCell id="n_hotfix" value="Hotfix path (urgent prod fix)&#xa;Branch hotfix/* from last prod tag vX.Y.Z" style="rounded=1;whiteSpace=wrap;html=1;strokeWidth=1;fillColor=#FFF4E5;strokeColor=#D79B00" vertex="1" parent="1">
          <mxGeometry x="60" y="240" width="280" height="90" as="geometry"/>
        </mxCell>

        <mxCell id="n2" value="Open PR &#x2192; main" style="rounded=1;whiteSpace=wrap;html=1;strokeWidth=1;fillColor=#E8F0FE;strokeColor=#6C8EBF" vertex="1" parent="1">
          <mxGeometry x="60" y="350" width="280" height="60" as="geometry"/>
        </mxCell>

        <!-- PR governance lane nodes -->
        <mxCell id="n3" value="Required PR checks (examples)&#xa;- Build + unit tests + coverage&#xa;- Lint/format&#xa;- SAST/dep scan/secret scan&#xa;- Policy checks (CODEOWNERS paths)" style="rounded=1;whiteSpace=wrap;html=1;strokeWidth=1;fillColor=#EAF9F0;strokeColor=#3C8D5A" vertex="1" parent="1">
          <mxGeometry x="410" y="150" width="300" height="140" as="geometry"/>
        </mxCell>

        <mxCell id="n4" value="Approvals + CODEOWNERS&#xa;No direct pushes to main&#xa;No force-push / history protected" style="rounded=1;whiteSpace=wrap;html=1;strokeWidth=1;fillColor=#EAF9F0;strokeColor=#3C8D5A" vertex="1" parent="1">
          <mxGeometry x="410" y="310" width="300" height="100" as="geometry"/>
        </mxCell>

        <mxCell id="n5" value="Merge PR &#x2192; main" style="rounded=1;whiteSpace=wrap;html=1;strokeWidth=1;fillColor=#EAF9F0;strokeColor=#3C8D5A" vertex="1" parent="1">
          <mxGeometry x="410" y="430" width="300" height="60" as="geometry"/>
        </mxCell>

        <!-- CI/CD + environments lane nodes -->
        <mxCell id="n6" value="On push to main&#xa;Build + Test&#xa;Publish artifact (immutable digest)&#xa;&quot;Build once, deploy many&quot;" style="rounded=1;whiteSpace=wrap;html=1;strokeWidth=1;fillColor=#F3E8FF;strokeColor=#9673A6" vertex="1" parent="1">
          <mxGeometry x="790" y="150" width="300" height="120" as="geometry"/>
        </mxCell>

        <mxCell id="n7" value="Auto Deploy &#x2192; DEV&#xa;GitHub Environment: dev&#xa;Allowed from: main" style="rounded=1;whiteSpace=wrap;html=1;strokeWidth=1;fillColor=#F3E8FF;strokeColor=#9673A6" vertex="1" parent="1">
          <mxGeometry x="1110" y="150" width="220" height="90" as="geometry"/>
        </mxCell>

        <mxCell id="n8" value="Create RC tag&#xa;vX.Y.Z-rc.N&#xa;(immutable tags via rulesets)" style="rounded=1;whiteSpace=wrap;html=1;strokeWidth=1;fillColor=#F3E8FF;strokeColor=#9673A6" vertex="1" parent="1">
          <mxGeometry x="790" y="290" width="300" height="90" as="geometry"/>
        </mxCell>

        <mxCell id="n9" value="Deploy &#x2192; PREPROD&#xa;From tags: v*-rc.*&#xa;Use same artifact digest" style="rounded=1;whiteSpace=wrap;html=1;strokeWidth=1;fillColor=#F3E8FF;strokeColor=#9673A6" vertex="1" parent="1">
          <mxGeometry x="1110" y="270" width="220" height="110" as="geometry"/>
        </mxCell>

        <mxCell id="n10" value="Create Release tag&#xa;vX.Y.Z" style="rounded=1;whiteSpace=wrap;html=1;strokeWidth=1;fillColor=#F3E8FF;strokeColor=#9673A6" vertex="1" parent="1">
          <mxGeometry x="790" y="410" width="300" height="70" as="geometry"/>
        </mxCell>

        <mxCell id="n11" value="Prod Gate&#xa;ServiceNow Change Request&#xa;- Create/Update CR&#xa;- Await CAB approval" style="rounded=1;whiteSpace=wrap;html=1;strokeWidth=1;fillColor=#FFE6E6;strokeColor=#B85450" vertex="1" parent="1">
          <mxGeometry x="1110" y="400" width="220" height="100" as="geometry"/>
        </mxCell>

        <mxCell id="n12" value="Deploy &#x2192; PROD&#xa;From tags: v*.*.*&#xa;Approved in ServiceNow&#xa;Same artifact digest" style="rounded=1;whiteSpace=wrap;html=1;strokeWidth=1;fillColor=#F3E8FF;strokeColor=#9673A6" vertex="1" parent="1">
          <mxGeometry x="1110" y="520" width="220" height="110" as="geometry"/>
        </mxCell>

        <!-- Optional notes -->
        <mxCell id="note1" value="Traceability: Tag &#x2192; Commit SHA &#x2192; Workflow Run &#x2192; Artifact Digest &#x2192; Environment Deployment History" style="rounded=1;whiteSpace=wrap;html=1;strokeWidth=1;fillColor=#FFFFFF;strokeColor=#D0D0D0;align=left" vertex="1" parent="1">
          <mxGeometry x="40" y="660" width="1290" height="60" as="geometry"/>
        </mxCell>

        <!-- Edges -->
        <mxCell id="e1" style="endArrow=block;html=1;rounded=1;strokeWidth=2" edge="1" parent="1" source="n1" target="n2">
          <mxGeometry relative="1" as="geometry"/>
        </mxCell>

        <mxCell id="e_hotfix_to_pr" value="or" style="endArrow=block;html=1;rounded=1;strokeWidth=2" edge="1" parent="1" source="n_hotfix" target="n2">
          <mxGeometry relative="1" as="geometry"/>
        </mxCell>

        <mxCell id="e2" style="endArrow=block;html=1;rounded=1;strokeWidth=2" edge="1" parent="1" source="n2" target="n3">
          <mxGeometry relative="1" as="geometry"/>
        </mxCell>

        <mxCell id="e3" style="endArrow=block;html=1;rounded=1;strokeWidth=2" edge="1" parent="1" source="n3" target="n4">
          <mxGeometry relative="1" as="geometry"/>
        </mxCell>

        <mxCell id="e4" style="endArrow=block;html=1;rounded=1;strokeWidth=2" edge="1" parent="1" source="n4" target="n5">
          <mxGeometry relative="1" as="geometry"/>
        </mxCell>

        <mxCell id="e5" style="endArrow=block;html=1;rounded=1;strokeWidth=2" edge="1" parent="1" source="n5" target="n6">
          <mxGeometry relative="1" as="geometry"/>
        </mxCell>

        <mxCell id="e6" style="endArrow=block;html=1;rounded=1;strokeWidth=2" edge="1" parent="1" source="n6" target="n7">
          <mxGeometry relative="1" as="geometry"/>
        </mxCell>

        <mxCell id="e7" style="endArrow=block;html=1;rounded=1;strokeWidth=2" edge="1" parent="1" source="n7" target="n8">
          <mxGeometry relative="1" as="geometry"/>
        </mxCell>

        <mxCell id="e8" style="endArrow=block;html=1;rounded=1;strokeWidth=2" edge="1" parent="1" source="n8" target="n9">
          <mxGeometry relative="1" as="geometry"/>
        </mxCell>

        <mxCell id="e9" style="endArrow=block;html=1;rounded=1;strokeWidth=2" edge="1" parent="1" source="n9" target="n10">
          <mxGeometry relative="1" as="geometry"/>
        </mxCell>

        <mxCell id="e10" style="endArrow=block;html=1;rounded=1;strokeWidth=2" edge="1" parent="1" source="n10" target="n11">
          <mxGeometry relative="1" as="geometry"/>
        </mxCell>

        <mxCell id="e11" value="Approved" style="endArrow=block;html=1;rounded=1;strokeWidth=2" edge="1" parent="1" source="n11" target="n12">
          <mxGeometry relative="1" as="geometry"/>
        </mxCell>

        <!-- Loop for "waiting approval" -->
        <mxCell id="e12" value="Pending" style="endArrow=block;html=1;rounded=1;strokeWidth=2;dashed=1" edge="1" parent="1" source="n11" target="n11">
          <mxGeometry relative="1" as="geometry">
            <mxPoint x="1350" y="450" as="targetPoint"/>
            <mxPoint x="1350" y="410" as="sourcePoint"/>
          </mxGeometry>
        </mxCell>

      </root>
    </mxGraphModel>
  </diagram>
</mxfile>
