<?xml version="1.0" encoding="UTF-8"?>
<mxfile host="app.diagrams.net" agent="Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/142.0.0.0 Safari/537.36 Edg/142.0.0.0" version="29.2.3">
  <diagram id="branching-release-process" name="Branching + Release + ServiceNow Gate">
    <mxGraphModel dx="1418" dy="786" grid="1" gridSize="10" guides="1" tooltips="1" connect="1" arrows="1" fold="1" page="1" pageScale="1" pageWidth="1400" pageHeight="900" math="0" shadow="0">
      <root>
        <mxCell id="0" />
        <mxCell id="1" parent="0" />
        <mxCell id="t1" parent="1" style="text;html=1;strokeColor=none;fillColor=none;align=left;verticalAlign=top;fontSize=18;fontStyle=1" value="GitHub Branching Strategy + Tag-based Promotion (dev / preprod / prod) + ServiceNow Gate" vertex="1">
          <mxGeometry height="30" width="1200" x="40" y="20" as="geometry" />
        </mxCell>
        <mxCell id="h_dev" parent="1" style="rounded=1;whiteSpace=wrap;html=1;strokeWidth=1;fillColor=#F5F5F5;strokeColor=#D0D0D0;align=center;fontStyle=1" value="Developer" vertex="1">
          <mxGeometry height="50" width="320" x="40" y="70" as="geometry" />
        </mxCell>
        <mxCell id="h_pr" parent="1" style="rounded=1;whiteSpace=wrap;html=1;strokeWidth=1;fillColor=#F5F5F5;strokeColor=#D0D0D0;align=center;fontStyle=1" value="PR Governance (Rulesets / Checks)" vertex="1">
          <mxGeometry height="50" width="360" x="380" y="70" as="geometry" />
        </mxCell>
        <mxCell id="h_cd" parent="1" style="rounded=1;whiteSpace=wrap;html=1;strokeWidth=1;fillColor=#F5F5F5;strokeColor=#D0D0D0;align=center;fontStyle=1" value="CI/CD + Environments" vertex="1">
          <mxGeometry height="50" width="600" x="760" y="70" as="geometry" />
        </mxCell>
        <mxCell id="n1" parent="1" style="rounded=1;whiteSpace=wrap;html=1;strokeWidth=1;fillColor=#E8F0FE;strokeColor=#6C8EBF" value="Create branch&#xa;feature/* or bugfix/*" vertex="1">
          <mxGeometry height="70" width="280" x="60" y="150" as="geometry" />
        </mxCell>
        <mxCell id="n_hotfix" parent="1" style="rounded=1;whiteSpace=wrap;html=1;strokeWidth=1;fillColor=#FFF4E5;strokeColor=#D79B00" value="Hotfix path (urgent prod fix)&#xa;Branch hotfix/* from last prod tag vX.Y.Z" vertex="1">
          <mxGeometry height="90" width="280" x="60" y="240" as="geometry" />
        </mxCell>
        <mxCell id="n2" parent="1" style="rounded=1;whiteSpace=wrap;html=1;strokeWidth=1;fillColor=#E8F0FE;strokeColor=#6C8EBF" value="Open PR → main" vertex="1">
          <mxGeometry height="60" width="280" x="60" y="350" as="geometry" />
        </mxCell>
        <mxCell id="n3" parent="1" style="rounded=1;whiteSpace=wrap;html=1;strokeWidth=1;fillColor=#EAF9F0;strokeColor=#3C8D5A" value="Required PR checks (examples)&#xa;- Build + unit tests + coverage&#xa;- Lint/format&#xa;- SAST/dep scan/secret scan&#xa;- Policy checks (CODEOWNERS paths)" vertex="1">
          <mxGeometry height="140" width="300" x="410" y="150" as="geometry" />
        </mxCell>
        <mxCell id="n4" parent="1" style="rounded=1;whiteSpace=wrap;html=1;strokeWidth=1;fillColor=#EAF9F0;strokeColor=#3C8D5A" value="Approvals + CODEOWNERS&#xa;No direct pushes to main&#xa;No force-push / history protected" vertex="1">
          <mxGeometry height="100" width="300" x="410" y="310" as="geometry" />
        </mxCell>
        <mxCell id="n5" parent="1" style="rounded=1;whiteSpace=wrap;html=1;strokeWidth=1;fillColor=#EAF9F0;strokeColor=#3C8D5A" value="Merge PR → main" vertex="1">
          <mxGeometry height="60" width="300" x="410" y="430" as="geometry" />
        </mxCell>
        <mxCell id="n6" parent="1" style="rounded=1;whiteSpace=wrap;html=1;strokeWidth=1;fillColor=#F3E8FF;strokeColor=#9673A6" value="On push to main&#xa;Build + Test&#xa;Publish artifact (immutable digest)&#xa;&quot;Build once, deploy many&quot;" vertex="1">
          <mxGeometry height="120" width="300" x="790" y="150" as="geometry" />
        </mxCell>
        <mxCell id="n7" parent="1" style="rounded=1;whiteSpace=wrap;html=1;strokeWidth=1;fillColor=#F3E8FF;strokeColor=#9673A6" value="Auto Deploy → DEV&#xa;GitHub Environment: dev&#xa;Allowed from: main" vertex="1">
          <mxGeometry height="90" width="220" x="1110" y="150" as="geometry" />
        </mxCell>
        <mxCell id="n8" parent="1" style="rounded=1;whiteSpace=wrap;html=1;strokeWidth=1;fillColor=#F3E8FF;strokeColor=#9673A6" value="Create RC tag&#xa;vX.Y.Z-rc.N&#xa;(immutable tags via rulesets)" vertex="1">
          <mxGeometry height="90" width="300" x="790" y="290" as="geometry" />
        </mxCell>
        <mxCell id="n9" parent="1" style="rounded=1;whiteSpace=wrap;html=1;strokeWidth=1;fillColor=#F3E8FF;strokeColor=#9673A6" value="Deploy → PREPROD&#xa;From tags: v*-rc.*&#xa;Use same artifact digest" vertex="1">
          <mxGeometry height="110" width="220" x="1110" y="270" as="geometry" />
        </mxCell>
        <mxCell id="n10" parent="1" style="rounded=1;whiteSpace=wrap;html=1;strokeWidth=1;fillColor=#F3E8FF;strokeColor=#9673A6" value="Create Release tag&#xa;vX.Y.Z" vertex="1">
          <mxGeometry height="70" width="300" x="790" y="410" as="geometry" />
        </mxCell>
        <mxCell id="n11" parent="1" style="rounded=1;whiteSpace=wrap;html=1;strokeWidth=1;fillColor=#FFE6E6;strokeColor=#B85450" value="Prod Gate&#xa;ServiceNow Change Request&#xa;- Create/Update CR&#xa;- Await CAB approval" vertex="1">
          <mxGeometry height="100" width="220" x="1110" y="400" as="geometry" />
        </mxCell>
        <mxCell id="n12" parent="1" style="rounded=1;whiteSpace=wrap;html=1;strokeWidth=1;fillColor=#F3E8FF;strokeColor=#9673A6" value="Deploy → PROD&#xa;From tags: v*.*.*&#xa;Approved in ServiceNow&#xa;Same artifact digest" vertex="1">
          <mxGeometry height="110" width="220" x="1110" y="520" as="geometry" />
        </mxCell>
        <mxCell id="note1" parent="1" style="rounded=1;whiteSpace=wrap;html=1;strokeWidth=1;fillColor=#FFFFFF;strokeColor=#D0D0D0;align=left" value="Traceability: Tag → Commit SHA → Workflow Run → Artifact Digest → Environment Deployment History" vertex="1">
          <mxGeometry height="60" width="1290" x="40" y="660" as="geometry" />
        </mxCell>
        <mxCell id="e1" edge="1" parent="1" source="n1" style="endArrow=block;html=1;rounded=1;strokeWidth=2" target="n2">
          <mxGeometry relative="1" as="geometry" />
        </mxCell>
        <mxCell id="e_hotfix_to_pr" edge="1" parent="1" source="n_hotfix" style="endArrow=block;html=1;rounded=1;strokeWidth=2" target="n2" value="or">
          <mxGeometry relative="1" as="geometry" />
        </mxCell>
        <mxCell id="e2" edge="1" parent="1" source="n2" style="endArrow=block;html=1;rounded=1;strokeWidth=2" target="n3">
          <mxGeometry relative="1" as="geometry" />
        </mxCell>
        <mxCell id="e3" edge="1" parent="1" source="n3" style="endArrow=block;html=1;rounded=1;strokeWidth=2" target="n4">
          <mxGeometry relative="1" as="geometry" />
        </mxCell>
        <mxCell id="e4" edge="1" parent="1" source="n4" style="endArrow=block;html=1;rounded=1;strokeWidth=2" target="n5">
          <mxGeometry relative="1" as="geometry" />
        </mxCell>
        <mxCell id="e5" edge="1" parent="1" source="n5" style="endArrow=block;html=1;rounded=1;strokeWidth=2" target="n6">
          <mxGeometry relative="1" as="geometry" />
        </mxCell>
        <mxCell id="e6" edge="1" parent="1" source="n6" style="endArrow=block;html=1;rounded=1;strokeWidth=2" target="n7">
          <mxGeometry relative="1" as="geometry" />
        </mxCell>
        <mxCell id="e7" edge="1" parent="1" source="n7" style="endArrow=block;html=1;rounded=1;strokeWidth=2" target="n8">
          <mxGeometry relative="1" as="geometry" />
        </mxCell>
        <mxCell id="e8" edge="1" parent="1" source="n8" style="endArrow=block;html=1;rounded=1;strokeWidth=2" target="n9">
          <mxGeometry relative="1" as="geometry" />
        </mxCell>
        <mxCell id="e9" edge="1" parent="1" source="n9" style="endArrow=block;html=1;rounded=1;strokeWidth=2" target="n10">
          <mxGeometry relative="1" as="geometry" />
        </mxCell>
        <mxCell id="e10" edge="1" parent="1" source="n10" style="endArrow=block;html=1;rounded=1;strokeWidth=2" target="n11">
          <mxGeometry relative="1" as="geometry" />
        </mxCell>
        <mxCell id="e11" edge="1" parent="1" source="n11" style="endArrow=block;html=1;rounded=1;strokeWidth=2" target="n12" value="Approved">
          <mxGeometry relative="1" as="geometry" />
        </mxCell>
        <mxCell id="e12" edge="1" parent="1" source="n11" style="endArrow=block;html=1;rounded=1;strokeWidth=2;dashed=1" target="n11" value="Pending">
          <mxGeometry relative="1" as="geometry">
            <mxPoint x="1350" y="410" as="sourcePoint" />
            <mxPoint x="1350" y="450" as="targetPoint" />
          </mxGeometry>
        </mxCell>
      </root>
    </mxGraphModel>
  </diagram>
</mxfile>
